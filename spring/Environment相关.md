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
# 常用实现类
StandardEnvironment:对应非servlet应用（新的webflux模块也是web，其environment的实现也是这个）
StandardServletEnvironment：对应servlet应用（spring5之前就是web应用），其继承结构如下：
![StandardServletEnvironment的继承结构](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTgwMzAyMjIwODI3NzUy?x-oss-process=image/format,png)

# 实现原理
关于profiles相关功能的实现其实很好理解，用一个Set保存所有active的profile就行了。
关键在于getProperty()相关功能的实现。
在spring的Environment中，引入了一个PropertySources用于表示PropertySource的集合，而PropertySource又代表这些表示属性的键值对的来源，比如：配置文件（properties文件）、map、ServletConfig等。
通常一个spring boot应用启动后，其environment会有如下的几个PropertySource：

| name                     |                             type                             |                             含义                             |
| ------------------------ | :----------------------------------------------------------: | :----------------------------------------------------------: |
| commandLineArgs          |               SimpleCommandLinePropertySource                |                          命令行参数                          |
| servletConfigInitParams  |                                                              |                      ServletConfig参数                       |
| servletContextInitParams |                                                              |                    ServletContext启动参数                    |
| spring.application.json  | SpringApplicationJsonEnvironmentPostProcessor.JsonPropertySource |                spring.application.json配置项                 |
| systemProperties         |                      MapPropertySource                       |               来源于System.getProperties()方法               |
| systemEnvironment        | SystemEnvironmentPropertySourceEPP.OriginAwareSystemEnvironmentPropertySource |                  来源于System.getEnv()方法                   |
| random                   |                  RandomValuePropertySource                   |           为random.int/long开头的属性赋一个随机值            |
| applicationConfig        |                OriginTrackedMapPropertySource                | 配置文件对应的propertySource，根据activeProfiles的情况，可能有多个 |

对于属性查找，environment会按照顺序逐个在propertySource中查找直到找到位置