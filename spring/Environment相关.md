## 基本介绍

Environment是spring中一个非常核心的接口，是spring属性加载的基础，代表当前应用运行的环境。
可以分为两个方面：

* profiles：设置profiles
* properties：查找属性配置

## 接口定义

```java
public interface Environment extends PropertyResolver {
   
   String[] getActiveProfiles();

   String[] getDefaultProfiles();

   boolean acceptsProfiles(String... profiles);

}
```
可以发现Environment接口中只有profiles相关的方法，因为其查找属性的功能由其父接口PropertyResolver提供：
```java
public interface PropertyResolver {

        boolean containsProperty(String key);
        String getProperty(String key);
        String getProperty(String key, String defaultValue);
        <T> T getProperty(String key, Class<T> targetType);
        <T> T getProperty(String key, Class<T> targetType, T defaultValue);
        String getRequiredProperty(String key) throws IllegalStateException;
        <T> T getRequiredProperty(String key, Class<T> targetType) throws IllegalStateException;
        String resolvePlaceholders(String text);
        String resolveRequiredPlaceholders(String text) throws IllegalArgumentException;

    }
```
## 常用实现类

StandardEnvironment:对应非servlet应用（新的webflux模块也是web，其environment的实现也是这个）
StandardServletEnvironment：对应servlet应用（spring5之前就是web应用）

StandardReactiveWebEnvironment:当使用web-reactive模块时，用的是这个实现（其实这个就是单纯继承了StandardEnvironment，并没有啥不一样，可能作者是留作以后扩展）

类的继承关系图谱：

<img src="/Users/didi/workspace/study/my-note/pic/StandardServletEnvironment.png" alt="StandardServletEnvironment" style="zoom:50%;" />

## 涉及到的类

* PropertySource:用于保存各个来源的配置项(总共有20个类)
* PropertySources：组合多个PropertySource(总共有2个类)
* PropertyResolver:具体执行属性查找的类，内部主要也是通过PropertySources来实现功能(总共有4个类)
* PropertyPlaceholderHelper：解析嵌套的配置属性的工具类(总共有1个类)
* Environment:(总共有9个类)
* ConversionService：如果找到的属性类型与需要的类型不匹配，就会用ConversionService来转化为需要的类型
* EnvironmentCapable：指明对象包含了一个Environment并暴露了获取Environment的接口,所以的ApplicationContext都实现了这个接口(总共有1个类)
* PropertySourceLoader:spring boot中用于加载属性配置文件(总共有3个类)
* PropertiesLoaderUtils：加载xml属性配置文件的工具类(总共有1个类)
* PropertySourceFactory:处理@PropertySource、@PropertySources注解(总共有4个类)

## 实现原理

关于profiles相关功能的实现其实很好理解，用一个Set保存所有active的profile就行了。
关键在于getProperty()相关功能的实现。
在spring的Environment中，引入了一个PropertySources用于表示PropertySource的集合，而PropertySource又代表这些表示属性的键值对的来源，比如：配置文件（properties文件）、map、ServletConfig等。
通常一个spring boot应用启动后，其environment会有如下的几个PropertySource：

| name                     |                             type                             |                             含义                             |
| ------------------------ | :----------------------------------------------------------: | :----------------------------------------------------------: |
| commandLineArgs          |               SimpleCommandLinePropertySource                |                          命令行参数                          |
| servletConfigInitParams  |                 ServletContextPropertySource                 |                      ServletConfig参数                       |
| servletContextInitParams |                 ServletConfigPropertySource                  |                    ServletContext启动参数                    |
| spring.application.json  | SpringApplicationJsonEnvironmentPostProcessor.JsonPropertySource |                spring.application.json配置项                 |
| systemProperties         |                      MapPropertySource                       |               来源于System.getProperties()方法               |
| systemEnvironment        | SystemEnvironmentPropertySourceEPP.OriginAwareSystemEnvironmentPropertySource |                  来源于System.getEnv()方法                   |
| random                   |                  RandomValuePropertySource                   |           为random.int/long开头的属性赋一个随机值            |
| applicationConfig        |                OriginTrackedMapPropertySource                | 配置文件对应的propertySource，根据activeProfiles的情况，可能有多个 |

### PropertySource的来源

systemProperties或者systemEnvironment对应的PropertySource都很容易获得，但是诸如application.properties配置文件对应的PropertySource是如何从一个普通的配置文件变为PropertySource的呢？

答案是通过PropertySourceLoader加载而来 or 通过@PropertySource注解

#### PropertySourceLoader

关于PropertySourceLoader：

```java
public interface PropertySourceLoader {
	//文件扩展名，比如properties，yaml
	String[] getFileExtensions();
	List<PropertySource<?>> load(String name, Resource resource) throws IOException;
}
```

这个接口有两个子类：

* PropertiesPropertySourceLoader：加载.properties文件，以及.xml文件
* YamlPropertySourceLoader：加载.yaml文件

#### @PropertySource

当spring在解析配置文件时，如果发现了@PropertySource注解，则会通过PropertySourceFactory的默认实现类DefaultPropertySourceFactory去加载@PropertySource中指定的属性文件，将其解析成ResourcePropertySource

### 方法执行过程分析

```java
//不论已那种getProperty的重载为入口，终究会走进这个方法
protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
		if (this.propertySources != null) {
      //按照顺序，遍历所有的PropertySource
			for (PropertySource<?> propertySource : this.propertySources) {
				if (logger.isTraceEnabled()) {
					logger.trace("Searching for key '" + key + "' in PropertySource '" +
							propertySource.getName() + "'");
				}
				Object value = propertySource.getProperty(key);
				if (value != null) {
          //如果找到了
					if (resolveNestedPlaceholders && value instanceof String) {
            //解析嵌套的配置项
						value = resolveNestedPlaceholders((String) value);
					}
          //打一个debug日志
					logKeyFound(key, propertySource, value);
          //转换为需要的类型
					return convertValueIfNecessary(value, targetValueType);
				}
			}
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Could not find key '" + key + "' in any property source");
		}
		return null;
	}
```

可以发现，其实，Environment的实现类，自己是没有实现getProperty()方法的，查找属性相关的方法都是由PropertySourcesPropertyResolver来具体实现的，其实现原理大致上可以描述为：

* 将PropertySource组装成PropertySources对象，这个PropertySources接口其实就是一系列PropertySource的集合
* 挨个找PropertySource中的配置项
* 如果找到了，再看这个配置项的value里面是不是有嵌套的配置项，比如你可能在配置文件中配置了：a.b=aaa${server.port}bbb这种，这是需要去解析这些嵌套的配置项
* 因为getProperty()有个重载的方法，需要返回指定的类型，所以如果进过之前的步骤找到的配置项类型不满足，会调用ConversionService来转化为需要的类型

## 其他的关联使用

### EnvironmentPostProcessor

定义：

```java
@FunctionalInterface
public interface EnvironmentPostProcessor {
  //当ApplicationEnvironmentPreparedEvent事件发生时，执行方法
	void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application);
}
```

EnvironmentPostProcessor是spring的一个扩展点，如果想在Environment创建好时，进行一些处理，可以通过自定义一个EnvironmentPostProcessor来实现。

做法：

* 创建一个META-INF/spring.factories文件
* 新增一个配置项org.springframework.boot.env.EnvironmentPostProcessor=你的具体实现类的全路径类名

spring boot中的已有实现：

* CloudFoundryVcapEnvironmentPostProcessor
* SystemEnvironmentPropertySourceEnvironmentPostProcessor:将propertySources中的systemEnvironment从SystemEnvironmentPropertySource替换为内部类OriginAwareSystemEnvironmentPropertySource
* SpringApplicationJsonEnvironmentPostProcessor:解析spring.application.json/SPRING_APPLICATION_JSON,并将其作为一个map PropertySource添加到Environment中
* ConfigFileApplicationListener
* DebugAgentEnvironmentPostProcessor：reactor相关，没看
* SpringBootTestRandomPortEnvironmentPostProcessor：测试用

## 结语

Environment及PropertySource，总共涉及了52个类左右，四舍五入，又是千里之行的10里路了！

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。如有错误，请指正~)