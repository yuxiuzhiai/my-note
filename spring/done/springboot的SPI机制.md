## 概念

spring boot启动的时候，就算你自己啥配置都没，它自己也会有很多已经配置好了的bean。我们知道在spring中，可以通过在xml中配置，或者通过@Component+@ComponentScan注解，或者通过@Configuration/@Bean注解来达到注册Bean到ApplicationContext中的效果，那这里你明明啥都没做，为啥已经有了这么多的bean了呢？

答：spring boot的某种类似SPI的机制

## 实现

通过SpringFactoriesLoader读取classpath低下的META-INF/spring.factories文件，这个文件可以有多个。

```java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
		//判断缓存，是否已经加载过了。
  	//对于每个ClassLoader，只会执行一次具体的加载过程，其他时候都是可以直接读cache的
  	MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		try {
      //加载META-INF/spring.factories文件
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
        //加载文件
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
        //遍历所有的key
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					String factoryTypeName = ((String) entry.getKey()).trim();
          //因为每个key可以对应多个用逗号分隔的值，因此也需要foreach一下
					for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
						//result是spring中自定义的Map实现，其实也没啥，就是value变成了一个List
            result.add(factoryTypeName, factoryImplementationName.trim());
					}
				}
			}
      //cache是以ClassLoader作为key的
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
```

大体上流程就是：

* 判断cache，如果有，直接读cache，返回
* 读取classpath低下的META-INF/spring.factories文件
* 遍历每个文件，遍历每个key，遍历每个逗号分隔的value
* 组合成一个Map，其中key是接口的全称类路径，value是这个key对应的实现的全称类路径的集合
* 将这个Map已ClassLoader为key，缓存

## 结语

在spring中，spring.factories文件是很重要的。所以虽然很简单，但肯定要知道

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。
如有错误，请指正~)