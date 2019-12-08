##基本环境

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
## spring boot 应用的启动

```java
/**
 * @author pk
 * @date 2018/02/22
 */
@SpringBootApplication
public class StudySpringApplication {

    public static void main(String[] args) {
        SpringApplication springApplication = new SpringApplication(StudySpringApplication.class);
        //可以在run之前,对AC进行一些自定义的配置,添加点ApplicationListener,ApplicationContextInitializer啥的
        springApplication.run(args);
    }
}
```
### 一.SpringApplication的初始化

代码如下：
```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));//1
		this.webApplicationType = deduceWebApplicationType();//2
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));//3
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));//4
		this.mainApplicationClass = deduceMainApplicationClass();//5
	}
```
关于ResourceLoader的介绍

#### 1.设置启动类primarySources到对应属性

	这个启动类就是在初始化SpringApplication时候的参数,可以有多个
### 2.获取web应用类型
根据当前classpath下存在的类来判断的:

 * 如果存在org.springframework.web.reactive.DispatcherHandler,就是REACTIVE
 * 如果不存在org.springframework.web.servlet.DispatcherServlet或者javax.servlet.Servlet,就是NONE
 * 否则,就是SERVLET
### 3.设置ApplicationContextInitializer
代码如下:
```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();//3.1
		Set<String> names = new LinkedHashSet<>(
				SpringFactoriesLoader.loadFactoryNames(type, classLoader));//3.2
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);//3.3
		AnnotationAwareOrderComparator.sort(instances);//3.4
		return instances;
	}
```
#### 3.1 获取线程上下文ClassLoader
TODO（ClassLoader的相关解释）
#### 3.2 使用SpringFactoriesLoader加载出classpath下所有路径为META-INF/spring.factories的资源文件,并读取key为ApplicationContextInitializer的值
会得到如些几个类（实际是全称类名）：

* SharedMetadataReaderFactoryContextInitializer
* AutoConfigurationReportLoggingInitializer
* ConfigurationWarningsACI
* ContextIdACI
* DelegatingACI
* ServerPortInfoACI
#### 3.3 利用反射创建出上述ACI的实例
#### 3.4 排序
根据Ordered接口/@PriorityOrder注解/@Order注解去排
其他地方的排序基本也是按照这个规则来排的
### 4.设置listener
跟设置ACI的方法一样,也是读取spring.factories文件中的对应key项所对应的所有类名,然后实例化、排序.
得到如下几个ApplicationListener:

* BackgroundPreinitializer
* ClearCachesAL
* ParentContextCloserAL
* FileEncodingAL
* AnsiOutputAL
* ConfigFileAL
* DelegatingAL
* ClasspathLoggingAL
* LoggingAL
* LiquibaseServiceLocatorAL

### 5.设置mainApplicationClass
就是找到main方法所在的类.是唯一的,跟上面的primarySources不是一回事

## 二.SpringApplication的run(String ... args)方法
代码如下:
```java
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();//1
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();//2
		SpringApplicationRunListeners listeners = getRunListeners(args);//3
		listeners.starting();//4
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);//5
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);//6
			configureIgnoreBeanInfo(environment);//7
			Banner printedBanner = printBanner(environment);//8
			context = createApplicationContext();//9
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);//10
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);//11
			refreshContext(context);//12
			afterRefresh(context, applicationArguments);//13
			listeners.finished(context, null);//14
			stopWatch.stop();
			//15
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			callRunners(context, applicationArguments);//16
			return context;
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, exceptionReporters, ex);
			throw new IllegalStateException(ex);
		}
	}
```
### 1. 创建一个StopWatch用于记录启动的时间
### 2. 配置java.awt.headless属性,默认为true
这个headless是一种模式,是指缺少显示屏、键盘或者鼠标时的系统配置
我找了spring和spring boot的源码,只有ImageBanner里面用到了
### 3.从spring.factories中加载出SpringApplicationRunListener,并构建一个SpringApplicationRunListeners
StringApplicationListeners就是对对个SpringApplicationRunListener的包装
现在看只有一个具体实现类

* EventPublishingRunListener

虽然这个类叫做listener,但其实起的作用是传播事件,更像是一个ApplicationEventMulticaster。
spring boot启动过程中的事件,主要由这个类来传播
### 4.发布一个ApplicationStartingEvent事件
关于spring事件发布机制，请看[spring事件机制](http://blog.csdn.net/yuxiuzhiai/article/details/79406558%20spring)
响应的AL

* LoggingAL：初始化日志系统
	做法是按照logback>log4j>jul的顺序查找classpath下存在的相关类
* LiquibaseServiceLocatorAL：实在是没有用过这个东西，不知道这个LiquibaseService是个啥...
### 5.根据启动时的命令行参数构建一个ApplicationArguments
ApplicationArguments的接口定义:
```java
public interface ApplicationArguments {

	String[] getSourceArgs();
	//--开头的参数
	Set<String> getOptionNames();

	boolean containsOption(String name);
	//--开头的参数是允许重复定义的，不会覆盖	
	List<String> getOptionValues(String name);

	List<String> getNonOptionArgs();

}
```
这个类的作用就是解析所有的启动参数的

### 6.prepareEnvironment():创建并配置environment
代码如下：
```java
private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		ConfigurableEnvironment environment = getOrCreateEnvironment();//6.1
		configureEnvironment(environment, applicationArguments.getSourceArgs());//6.2
		listeners.environmentPrepared(environment);//6.3
		bindToSpringApplication(environment);//6.4
		if (this.webApplicationType == WebApplicationType.NONE) {
			environment = new EnvironmentConverter(getClassLoader())
					.convertToStandardEnvironmentIfNecessary(environment);
		}
		ConfigurationPropertySources.attach(environment);//6.5
		return environment;
	}
```
#### 6.1 根据WebApplicationType创建对应类型的environment
关于spring中Environment/PropertySource的介绍:

根据webApplicationType：

*  SERVLET -> StandardServletEnvironment
* NONE/REACTIVE -> StandardEnvironment

初始化的时候会在构造函数中调用customizePropertySources()方法，其结果是会在environment的propertySources属性的propertySourcesList列表中加入以下PropertySource：

 * 代表ServletConfig的构建参数的，名为servletConfigInitParams的StubPropertySource（目前只是作为占位符存在）
 * 代表ServletContext的构建参数的，名为servletContextInitParams的StubPropertySource（目前只是作为占位符存在）
 * 代表jvm属性参数的名为systemProperties的MapPropertySource（来源于System.getProperties()方法）
 * 代表环境变量的名为systemEnvironment的SystemEnvironmentPropertySource（来源于System.getEnv()方法）
#### 6.2 配置environment

 * 配置PropertySources
	* 看看SpringApplication的defaultProperties属性是否为空，如果不为空则加入一个MapPropertySource到propertySources中。这个defaultProperties属性可以在SpringApplication调用run方法之前通过setDefaultProperties()方法设置
	* 看看有没有命令号参数，如果有则创建一个SimpleCommandLinePropertySource加入到propertySources中
 * 配置activeProfiles:就是将spring.profiles.active属性的值作为数组读入
#### 6.3 listeners发布一个ApplicationEnvironmentPreparedEvent
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
#### 6.4 将environment绑定到SpringApplication中
todo...
#### 6.5 往propertySources中加入一个ConfigurationPropertySourcesPropertySource
### 7.配置一个spring.beaninfo.ignore属性,默认为true
### 8.打印banner
### 9.根据webApplicationType构建ApplicationContext

 * SERVLET -> AnnotationConfigServletWebServerAC
 * REACTIVE -> ReactiveWebServerAC
 * NONE -> AnnotationConfigAC
按照继承结构：

* DefaultResourceLoader:ResourceLoader的默认实现，作用就是加载Resource，内部有个比较重要的ProtocolResolver集合属性，具体使用请看[ProtocolResolver解析](http://blog.csdn.net/yuxiuzhiai/article/details/79080154)
*  AbstractAC：配置了一个ResourcePatternResolver，具体请看[Resource相关](http://blog.csdn.net/yuxiuzhiai/article/details/79117948)
* GenericAC：初始化了一个BeanFactory，具体类型是DefaultListableBeanFactory