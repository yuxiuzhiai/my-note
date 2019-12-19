# 概念



# 用法

用一个十分简单的例子来演示

```java
public class ApplicationContextTest {
  @Test
  public void testSimpleLoad() {
    AnnotationConfigWebApplicationContext ac = new AnnotationConfigWebApplicationContext();
    ac.register(MyConfig.class);
    //继承了AbstractRefreshableApplicationContext的AC实现，在使用前需要调一下refresh()方法
    ac.refresh();
    UserInfo userInfo = ac.getBean(UserInfo.class);
    assertEquals("a", userInfo.getA());
  }
}
```

MyConfig.java

```java
@Configuration
public class MyConfig {
  @Bean(name = "/foo")
  public UserInfo userInfo(){
    UserInfo userInfo = new UserInfo();
    userInfo.setA("a");
    userInfo.setB("b");
    return userInfo;
  }
}
```



# 实现

创建一个AC实现，然后注册@Configuration注解的配置文件（关于解析@Configuration配置文件，请见https://blog.csdn.net/yuxiuzhiai/article/details/103589673）

然后调用refresh()方法刷新容器。

在spring中，如果一个ApplicationContext实现类没有继承AbstractRefreshableApplicationContext，则必然是在创建实例的构造方法内就调用了refresh()方法，而且在整个生命周期只会调那么一次；如果实现了AbstractRefreshableApplicationContext，则一般不会在创建AC时就调用refresh()方法，会等着你注册好各种BeanDefinition的来源，然后由自己主动去调用refresh()方法，而且refresh()方法可以调用多次。

本篇仅关注容器刷新refresh()方法。

即这个方法：(去掉了try catch)

```java
public void refresh() throws BeansException, IllegalStateException {
  //1.准备
  prepareRefresh();

  //2.新建BeanFactory
  ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

  //3.配置BeanFactory
  prepareBeanFactory(beanFactory);


  // Allows post-processing of the bean factory in context subclasses.
  postProcessBeanFactory(beanFactory);

  // Invoke factory processors registered as beans in the context.
  invokeBeanFactoryPostProcessors(beanFactory);

  // Register bean processors that intercept bean creation.
  registerBeanPostProcessors(beanFactory);

  // Initialize message source for this context.
  initMessageSource();

  // Initialize event multicaster for this context.
  initApplicationEventMulticaster();

  // Initialize other special beans in specific context subclasses.
  onRefresh();

  // Check for listener beans and register them.
  registerListeners();

  // Instantiate all remaining (non-lazy-init) singletons.
  finishBeanFactoryInitialization(beanFactory);

  // Last step: publish corresponding event.
  finishRefresh();
}
```

接下来，挨个分析每个方法：

## 1.准备

进入方法(删除了不太重要的部分)

```java
protected void prepareRefresh() {

  //1.1.添加一些自己的PropertySource
  initPropertySources();

  //1.2.验证必须属性
  getEnvironment().validateRequiredProperties();
}
```

1.1和1.2都是用于子类扩展的。1.1用于添加一些特定ApplicationContext的PropertySource，比如Servlet相关的ApplicationContext实现就会在这一步将ServletContext转换为PropertySource加到Environment里面去；1.2是用于验证一些必须的属性是否存在

## 2.新建BeanFactory

这一步是真正创建BeanFactory的地方，进入方法

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
  refreshBeanFactory();
  return getBeanFactory();
}
```

主要的逻辑在refreshBeanFactory()方法

```java
protected final void refreshBeanFactory() throws BeansException {
  //先判断是不是已经存在BeanFactory，如果存在，先关闭
  if (hasBeanFactory()) {
    destroyBeans();
    closeBeanFactory();
  }
  try {
    //新建一个BeanFactory，如果存在父容器，就把父BeanFactory设置进去
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    beanFactory.setSerializationId(getId());
    //设置一些自己配置的属性，比如是否允许覆盖，是否允许循环依赖
    customizeBeanFactory(beanFactory);
    //加载BeanDefinition，有@Configuration注解文件和扫描指定包下面的@Component注解两种方法
    //如果是xml相关的ApplicationContext实现，就会是解析xml的那一套
    loadBeanDefinitions(beanFactory);
    synchronized (this.beanFactoryMonitor) {
      this.beanFactory = beanFactory;
    }
  }
  catch (IOException ex) {
    throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
  }
}
```

加载BeanDefinition这一步，根据不同类型的ApplicationContext，大体上有三种方式：

* 解析@Configuration配置文件。请见https://blog.csdn.net/yuxiuzhiai/article/details/103589673
* 扫描指定包下的@Component注解。请见https://blog.csdn.net/yuxiuzhiai/article/details/103589648
* 解析xml配置文件。请见https://blog.csdn.net/yuxiuzhiai/article/details/103541347

## 3.配置BeanFactory

创建好BeanFactory，加载了BeanDefinition之后，BeanFactory还是很裸的，有很多属性还没有值，很多功能还不具备，所以就有了配置BeanFactory这一步，来完善BeanFactory，方便后面的使用。进入方法：

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  //设置classLoader
  beanFactory.setBeanClassLoader(getClassLoader());
  //设置spel处理器，添加spel的支持
  beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
  //PropertyEditor相关
  beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

  // Configure the bean factory with context callbacks.
  beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
  beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
  beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
  beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
  beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
  beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
  beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

  // BeanFactory interface not registered as resolvable type in a plain factory.
  // MessageSource registered (and found for autowiring) as a bean.
  beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
  beanFactory.registerResolvableDependency(ResourceLoader.class, this);
  beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
  beanFactory.registerResolvableDependency(ApplicationContext.class, this);

  // Register early post-processor for detecting inner beans as ApplicationListeners.
  beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

  //增加aspectJ的支持
  if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
    beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
    // Set a temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
  }

  //注册Environment、系统属性对应的PropertySource、jvm属性对应的PropertySource
  if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
    beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
  }
  if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
    beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
  }
  if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
    beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
  }
}
```

blabla

## 4.BeanFactory的后处理



# 结语

在spring中，spring.factories文件是很重要的。所以虽然很简单，但肯定要知道

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。如有错误，请指正~)