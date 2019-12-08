## 概念

在spring中，Resource对应一个资源，这个资源可以是一个文件，可以对应一个URL，可以是classpath低下的一个配置文件。

在spring中，Resource由ResourceLoader加载。说到ResourceLoader大家可能不熟，但是著名的ApplicationContext接口就是继承了ResourceLoader接口，所以具备加载Resource的能力

##常见用法

```java
package com.example.demo;

import org.junit.jupiter.api.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.core.io.Resource;

import java.io.IOException;
import java.util.Arrays;

/**
 * @author pengkai
 * @date 2019-12-08
 */
public class ResourceTest {

    @Test
    public void test() throws IOException {
        //当ResourceLoader用
        ApplicationContext ac = new AnnotationConfigApplicationContext();
        //其实返回的并不是ClassPathResource,而是DefaultResourceLoader$ClassPathContextResource
        //如果没有特定协议前缀的话,对于不同的ApplicationContext实现,这里的处理也不一样
        //如果上面的AC实现换成FileSystemXmlApplicationContext,那就会把b.txt当成相对于项目目录的相对路径来处理
        Resource classpathResource = ac.getResource("b.txt");
        //返回UrlResource
        Resource urlResource = ac.getResource("http://www.baidu.com");
        //返回FileUrlResource(本地的这个绝对路径下要用这个文件不然报错)
        Resource fileResource = ac.getResource("file:/logs/controller.log");
        //返回ClassPathResource[]
        Resource[] resources = ac.getResources("classpath*:b.txt");

        printResource(classpathResource);

        printResource(urlResource);

        printResource(fileResource);

        Arrays.stream(resources).forEach(ResourceTest::printResource);
    }

    private static void printResource(Resource resource) {
        System.out.println("start resource print...");
        System.out.println("class:" + resource.getClass().getName());
        System.out.println("exists:" + resource.exists());
        System.out.println("isReadable:" + resource.isReadable());
        System.out.println("isOpen:" + resource.isOpen());
        System.out.println("isFile:" + resource.isFile());
        try {
            System.out.println("getURL:" + resource.getURL());
            System.out.println("getURI:" + resource.getURI());
            System.out.println("getFile:" + resource.getFile());
            System.out.println("contentLength:" + resource.contentLength());
            System.out.println("lastModified:" + resource.lastModified());
        } catch (IOException e) {
            e.printStackTrace();
        }

        System.out.println("getFilename:" + resource.getFilename());
        System.out.println("getDescription:" + resource.getDescription());
        System.out.println("end resource print...");
    }
}
```

## spring为啥要搞一套Resource接口出来

spring的回答是：对于访问底层的（原文是low-level）的资源文件，Resource接口较java.net.URL更能胜任。事实也的确是这样，用Resource/ResourceLoader这一套来处理资源文件的加载的确是很方便。下面就来逐步看看这个方便时如何实现的

## 相关类和具体实现的方法

### Resource接口定义

```java
public interface Resource extends InputStreamSource {

    boolean exists();
    default boolean isReadable() {
        return true;
    }
    default boolean isOpen() {
        return false;
    }
    //spring 5.0新增
    default boolean isFile() {
        return false;
    }
    URL getURL() throws IOException;
    URI getURI() throws IOException;
    File getFile() throws IOException;
    //5.0新增,每次调用都会返回一个新的Channel
    default ReadableByteChannel readableChannel() throws IOException {
        return Channels.newChannel(getInputStream());
    }
    long contentLength() throws IOException;
    long lastModified() throws IOException;
    Resource createRelative(String relativePath) throws IOException;
    String getFilename();
    String getDescription();

}
```
### 继承的接口：

```java
public interface InputStreamSource {
	InputStream getInputStream() throws IOException;
}
```
### 子类/子接口：

WritableResource：子接口，增加写入资源文件的能力。FileUrlResource是其子类，对于"file:"开头的路径字符串，如果是由DefaultResourceLoader加载，则FileUrlResource是Resource的具体返回类型；

UrlResource：java.net.URL定位符的对应实现（如"http:"）

ClassPathResource:当前classpath下资源文件的对应实现（如"classpath:"）

ByteArrayResource/InputStreamResource:对应字符数组/InputStream的Resource实现

还有很多，就不列举了...

### 加载Resource的接口:

```java
public interface ResourceLoader {
	//就是"classpath"字符串,ResourceLoader的实现类并处理不了"classpath*",需要其子接口ResourcePatternResolver处理
	String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;
	//从一个表示路径的字符串得到Resource
	Resource getResource(String location);
	ClassLoader getClassLoader();
}
```
ResourceLoader的子接口：
```java
public interface ResourcePatternResolver extends ResourceLoader {
	String CLASSPATH_ALL_URL_PREFIX = "classpath*:";
	//可以用于解析ant风格的路径字符串
	Resource[] getResources(String locationPattern) throws IOException;
}
```
###ResourceLoader的实现类
:加载resource的能力提供者，各种ApplicationContext或者BeanFactory的getResource()方法，最终都是委派给这个类完成的

PathMatchingResourcePatternResolver(ResourcePatternResolver的实现类):提供了getResources()方法，各种AC/BF的getResources()方法，最终都是由这个类完成的

其实从类的继承关系上看，ApplicationContext及其实现，也是ResourceLoader的子类，只是没有在这里列举，为啥呢？因为ApplicationContext及其子类虽然都直接或者间接的实现ResourceLoader接口，但是观其代码，即可知道，他们其实并没有真的实现加载Resource的方法，只是在类的内部聚合了一个DefaultResourceLoader或者PathMatchingResourcePatternResolver，用于加载Resource，所以这里就没有提到ApplicationContext了。

##DefaultResourceLoader.getResource()方法的实现

```java
public Resource getResource(String location) {
	//location不能是null
    Assert.notNull(location, "Location must not be null");
	//检查有没有自定义的ProtocolResolver实现,关于ProtocolResolver，请看 http://blog.csdn.net/yuxiuzhiai/article/details/79080154
    for (ProtocolResolver protocolResolver : this.protocolResolvers) {
        Resource resource = protocolResolver.resolve(location, this);
        if (resource != null)
            return resource;
    }
    if (location.startsWith("/")) {
	    //不同的ApplicationContext实现是重写了这个方法的
        return getResourceByPath(location);
    }else if (location.startsWith(CLASSPATH_URL_PREFIX)) {//如果以classpath开头
        return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
    }else {
        try {
	        //将location字符串转成URL对象
            URL url = new URL(location);
            //isFileURL()就是判断是否以 "file", "vfsfile" ，"vfs"开头
            return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
        }
        catch (MalformedURLException ex) {
            //如果不是以上面的任何一种前缀开头,DefaultResourceLoader中是把没有任何特定前缀的location字符串当成classpath来处理的
            //不同的AC实现可能重写了这个方法,比如FileSystemXmlApplicationContext重写了这个方法,当成FileSystemResource处理,GenericWebApplicationContext则是当成ServletContextResource处理
            return getResourceByPath(location);
        }
    }
}
```
##PathMatchingResourcePatternResolver.getResources()的实现
```java
public Resource[] getResources(String locationPattern) throws IOException {
		Assert.notNull(locationPattern, "Location pattern must not be null");
		//如果以"classpath*:"开头
		if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
			//判断路径中去除classpath*后还有没有*或者?(ant中的通配符)
			if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) {
				//有classpath*,有通配符
				//@1
				return findPathMatchingResources(locationPattern);
			} else {
				//有classpath*,无通配符
				//@2
				return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));
			}
		} else {
			// war文件的处理,war以"*/"结尾
			int prefixEnd = (locationPattern.startsWith("war:") ? locationPattern.indexOf("*/") + 1 :
					locationPattern.indexOf(":") + 1);
			if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {
				//无classpath*,有通配符
				return findPathMatchingResources(locationPattern);
			} else {
				//无classpath*,无通配符,所以还是由DefaultResourceLoader处理
				return new Resource[] {getResourceLoader().getResource(locationPattern)};
			}
		}
	}
```
1中的方法：

 1. 将location分割成目录和文件名两个部分（两个部分都可能存在通配符）
 2. 将每个匹配的目录解析成Resource，就是源码中的rootDirResource
 3. 对于每个目录Resource，根据第一步得到的文件名去匹配，并且考虑了dootDirResource是vfs/jar/war/zip的情况
 4. 将所以匹配的文件归并到一个Resource数组中

 源码如下：
```java
protected Resource[] findPathMatchingResources(String locationPattern) throws IOException {
		//location中的目录部分,包含签名的协议(如:classpath*)
		String rootDirPath = determineRootDir(locationPattern);
		//location中的文件部分
		String subPattern = locationPattern.substring(rootDirPath.length());
		//所有匹配的目录
		Resource[] rootDirResources = getResources(rootDirPath);
		Set<Resource> result = new LinkedHashSet<>(16);
		for (Resource rootDirResource : rootDirResources) {
			rootDirResource = resolveRootDirResource(rootDirResource);
			URL rootDirUrl = rootDirResource.getURL();
			if (rootDirUrl.getProtocol().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) 
				result.addAll(VfsResourceMatchingDelegate.findMatchingResources(rootDirUrl, subPattern, getPathMatcher()));
			else if (ResourceUtils.isJarURL(rootDirUrl) || isJarResource(rootDirResource)) 
				result.addAll(doFindPathMatchingJarResources(rootDirResource, rootDirUrl, subPattern));
			else 
				result.addAll(doFindPathMatchingFileResources(rootDirResource, subPattern));
		}
		return result.toArray(new Resource[result.size()]);
	}
```
2处的方法：
```java
protected Resource[] findAllClassPathResources(String location) throws IOException {
		String path = location;
		if (path.startsWith("/")) 
			path = path.substring(1);
		//@3
		Set<Resource> result = doFindAllClassPathResources(path);
		return result.toArray(new Resource[result.size()]);
	}
```
3处的方法：
```java
protected Set<Resource> doFindAllClassPathResources(String path) throws IOException {
		Set<Resource> result = new LinkedHashSet<>(16);
		//使用ClassLoader中的getResources方法去加载classpath resource
		ClassLoader cl = getClassLoader();
		Enumeration<URL> resourceUrls = (cl != null ? cl.getResources(path) : ClassLoader.getSystemResources(path));
		while (resourceUrls.hasMoreElements()) {
			URL url = resourceUrls.nextElement();
			result.add(convertClassLoaderURL(url));
		}
		//代表一开始的location是classpath*:或者classpath*:/
		if ("".equals(path)) {
			//会将所依赖的所有jar文件本身和一些目录本身加载成Resource,添加到result中
			//但是不会继续解析jar文件中包含的文件或者是目录中的子文件
			addAllClassLoaderJarRoots(cl, result);
		}
		return result;
	}
```

##结语
Resource也算是spring中比较重要的部分了，因为吧spring5的源码总归有4000个java文件左右(排除测试相关)，Resource相关的就20多接近30个java文件了，四舍五入就站到了spring-framework源码的百分之一啊，千里之行，已经走了10里了啊！

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。
如有错误，请指正~)