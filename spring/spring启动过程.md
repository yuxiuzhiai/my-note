# 基本环境

开发工具:Intellij IDEA 2017（盗）版

java版本：1.8.0_151

spring的github地址:[spring-framework](https://github.com/spring-projects/spring-framework)

示例项目地址:[study-spring](https://github.com/yuxiuzhiai/study-spring)

准备：git clone或直接下载github上的spring源码，导入idea中，在项目路径下执行gradle build (如果本机没有gradle环境，或者版本差很多，就用gradlew代替)，会build很久，可以事先将阿里的maven仓库地址加到repositories中，像这样:

```gradle
repositories {
		maven {
            url "http://maven.aliyun.com/nexus/content/groups/public/"
        }
		maven { url "https://repo.spring.io/plugins-release" }
}
```
会用到的缩写:

* ApplicationContext -> AC
* ApplicationContextInitializer -> ACI
* BeanFactory -> BF 
* ApplicationListener -> AL
* EnvironmentPostProcessor -> EPP
# 使用

```java
/**
 * @author pk
 * @date 2018/02/22
 */
@SpringBootApplication
public class StudySpringApplication {

    public static void main(String[] args) {
      	//1.创建SpringApplication，进行一些配置项的设置，SPI机制加载spring.factories的配置项到缓存中
        SpringApplication springApplication = new SpringApplication(StudySpringApplication.class);
        //2.SpringApplication的启动
      	//可以在run之前,对AC进行一些自定义的配置,添加点ApplicationListener,ApplicationContextInitializer啥的
        springApplication.run(args);
    }
}
```
# 实现

## 1.SpringApplication的初始化

代码如下：
```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
  this.resourceLoader = resourceLoader;
  Assert.notNull(primarySources, "PrimarySources must not be null");
  this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
  this.webApplicationType = WebApplicationType.deduceFromClasspath();
  //加载spring.factories中的ApplicationContextInitializer配置项
  //关于spring boot中的这种SPI机制详情，请看https://blog.csdn.net/yuxiuzhiai/article/details/103480028
  setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
  //加载spring.factories中的ApplicationListener配置项
  setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
  this.mainApplicationClass = deduceMainApplicationClass();
}
```
比较简单，简单叙述一下过程：

* 设置启动类primarySources到对应属性。这个启动类就是在初始化SpringApplication时候的参数,可以有多个
* 获取web应用类型。根据当前classpath下存在的类来判断的:
   * 如果存在org.springframework.web.reactive.DispatcherHandler,就是REACTIVE
   * 如果不存在org.springframework.web.servlet.DispatcherServlet或者javax.servlet.Servlet,就是NONE
   * 否则,就是SERVLET

* 根据spring.factories文件加载ApplicationContextInitializer和ApplicationListener，并设置到对应的属性上去
* 设置mainApplicationClass。就是找到main方法所在的类.是唯一的,跟上面的primarySources不是一回事

关于spring boot中的这种SPI机制详情，请看https://blog.csdn.net/yuxiuzhiai/article/details/103480028

关于spring boot中的事件机制，请看https://blog.csdn.net/yuxiuzhiai/article/details/79406558

## 2.SpringApplication的启动

代码如下:
```java
public ConfigurableApplicationContext run(String... args) {
  //StopWatch用于统计启动时间;
  StopWatch stopWatch = new StopWatch();
  stopWatch.start();
  ConfigurableApplicationContext context = null;
  Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
  //headless的配置
  configureHeadlessProperty();
  //2.1.SpringApplicationRunListener，专门用于处理spring boot启动时的事件
  SpringApplicationRunListeners listeners = getRunListeners(args);
  //2.2.发布一个事件
  listeners.starting();
  try {
    //2.3.封装和处理命令行参数
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
    //2.4.创建并配置Environment
    ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
    configureIgnoreBeanInfo(environment);
    Banner printedBanner = printBanner(environment);
    //2.5.创建ApplicationContext
    context = createApplicationContext();
    exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                                                     new Class[] { ConfigurableApplicationContext.class }, context);
    prepareContext(context, environment, listeners, applicationArguments, printedBanner);
    refreshContext(context);
    afterRefresh(context, applicationArguments);
    stopWatch.stop();
    if (this.logStartupInfo) {
      new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
    }
    listeners.started(context);
    callRunners(context, applicationArguments);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, listeners);
    throw new IllegalStateException(ex);
  }

  try {
    listeners.running(context);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, null);
    throw new IllegalStateException(ex);
  }
  return context;
}
```
### 2.1.从spring.factories中加载出SpringApplicationRunListener,并构建一个SpringApplicationRunListeners

StringApplicationListeners就是对对个SpringApplicationRunListener的包装
现在看只有一个具体实现类

* EventPublishingRunListener

虽然这个类叫做listener,但其实起的作用是传播事件,更像是一个ApplicationEventMulticaster。
spring boot启动过程中的事件,主要由这个类来传播。

### 2.2.发布一个ApplicationStartingEvent事件

关于spring事件发布机制，请看[spring事件机制](http://blog.csdn.net/yuxiuzhiai/article/details/79406558%20spring)
响应的AL

* LoggingAL：初始化日志系统
	做法是按照logback>log4j>jul的顺序查找classpath下存在的相关类
* BackgroundPreinitializer:将一些耗时的组件放到后台初始化，比如：
  * ConversionServiceInitializer
  * ValidationInitializer
  * MessageConverterInitializer
  * JacksonInitializer
  * CharsetInitializer

### 2.3.根据启动时的命令行参数构建一个ApplicationArguments

具体请看：https://blog.csdn.net/yuxiuzhiai/article/details/103465149

### 2.4.创建并配置environment

代码如下：
```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,ApplicationArguments applicationArguments) {
  //2.4.1.根据当前应用类型创建一个对应的Environment
  ConfigurableEnvironment environment = getOrCreateEnvironment();
  //2.4.2.配置Environment的profile、conversionservice,并把命令行参数转换成PropertySource加到Environment中
  configureEnvironment(environment, applicationArguments.getSourceArgs());
  ConfigurationPropertySources.attach(environment);
  //2.4.3.发布ApplicationEnvironmentPreparedEvent事件
  listeners.environmentPrepared(environment);
  //2.4.4
  bindToSpringApplication(environment);
  if (!this.isCustomEnvironment) {
    environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,deduceEnvironmentClass());
  }
  //2.4.5
  ConfigurationPropertySources.attach(environment);
  return environment;
}
```
#### 2.4.1 根据WebApplicationType创建对应类型的environment

关于spring中Environment/PropertySource的介绍:https://blog.csdn.net/yuxiuzhiai/article/details/79427325

根据webApplicationType：

*  SERVLET -> StandardServletEnvironment
* NONE -> StandardEnvironment
* REACTIVE -> StandardReactiveWebEnvironment

初始化的时候会在构造函数中调用customizePropertySources()方法，其结果是会在environment的propertySources属性的propertySourcesList列表中加入以下PropertySource：

 * 代表ServletConfig的构建参数的，名为servletConfigInitParams的StubPropertySource（目前只是作为占位符存在）
 * 代表ServletContext的构建参数的，名为servletContextInitParams的StubPropertySource（目前只是作为占位符存在）
 * 代表jvm属性参数的名为systemProperties的MapPropertySource（来源于System.getProperties()方法）
 * 代表环境变量的名为systemEnvironment的SystemEnvironmentPropertySource（来源于System.getEnv()方法）
#### 2.4.2 配置environment

 * 设置ConversionService
 * 配置PropertySources
	* 看看SpringApplication的defaultProperties属性是否为空，如果不为空则加入一个MapPropertySource到propertySources中。这个defaultProperties属性可以在SpringApplication调用run方法之前通过setDefaultProperties()方法设置
	* 看看有没有命令行参数，如果有则创建一个SimpleCommandLinePropertySource加入到propertySources中
 * 配置activeProfiles:就是将spring.profiles.active属性的值作为数组读入
#### 2.4.3 listeners发布一个ApplicationEnvironmentPreparedEvent

这是一个很重要的事件，响应的listener有很多：

 * ConfigFileApplicationListener
	* 从spring.factories中读取所有的EnvironmentPostProcessor，它自己也是个EPP。
	* 排序，然后逐个调用
		* SystemEnvironmentPropertySourceEPP:替换systemEnvironment为其内部类OriginAwareSystemEnvironmentPropertySource
		* SpringApplicationJsonEPP：解析spring.application.json属性，作为JsonPropertySource加入propertySources中
		* CloudFoundryVcapEPP:VCAP相关，不懂，略过
		* ConfigFileAL（自己）：
			* 添加一个RandomValuePropertySource，作用是设置所有random.int/long开头的属性为一个随机值
			* 从spring.factories中读取PropertySourceLoader：
				* PropertiesPropertySourceLoader：读取.properties配置文件
				* YamlPropertySourceLoader：读取.yaml配置文件
			* 根据activeProfiles解析每个对应的配置文件成OriginTrackedMapPropertySource，加入到propertySources中
* LoggingApplicationListener：日志系统的初始化

#### 2.4.4 将environment绑定到SpringApplication中

todo...
#### 2.4.5 往propertySources中加入一个ConfigurationPropertySourcesPropertySource

### 2.5.根据webApplicationType构建ApplicationContext

 * SERVLET -> AnnotationConfigServletWebServerAC
 * REACTIVE -> ReactiveWebServerAC
 * NONE -> AnnotationConfigAC
按照继承结构：

* DefaultResourceLoader:ResourceLoader的默认实现，作用就是加载Resource，内部有个比较重要的ProtocolResolver集合属性，具体使用请看[ProtocolResolver解析](http://blog.csdn.net/yuxiuzhiai/article/details/79080154)
*  AbstractAC：配置了一个ResourcePatternResolver，具体请看[Resource相关](http://blog.csdn.net/yuxiuzhiai/article/details/79117948)
* GenericAC：初始化了一个BeanFactory，具体类型是DefaultListableBeanFactory