# 概念



# 用法



# 实现

在spring 中，aop相关的一个最重要的注解就是@Aspect了，代表当前bean是一个aop的切面。而具体处理这个注解的，也是spring中实现动态aop最为关键的类就是AnnotationAwareAspectJAutoProxyCreator。

先看下AnnotationAwareAspectJAutoProxyCreator的类继承图：

<img src="/Users/didi/workspace/study/my-note/pic/AspectJAwareAdvisorAutoProxyCreator.png" style="zoom:80%;" />

值得关注的是其继承了BeanPostProcessor。动态aop代理的创建也是在BeanPostProcessor接口的方法postProcessAfterInitialization中实现的。直接进入方法：

```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
  if (bean != null) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    if (this.earlyProxyReferences.remove(cacheKey) != bean) {
      //核心逻辑
      return wrapIfNecessary(bean, beanName, cacheKey);
    }
  }
  return bean;
}
```

最为关键的代码就是:wrapIfNecessary(bean, beanName, cacheKey)。进入：

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
  //以下都是排除一些不需要代理的bean，基础设施bean啊，指定不需要代理的bean啊之类的
  if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
    return bean;
  }
  if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
    return bean;
  }
  if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
  }

  //1.第一个核心子方法
  Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
  if (specificInterceptors != DO_NOT_PROXY) {
    this.advisedBeans.put(cacheKey, Boolean.TRUE);
    Object proxy = createProxy(
      bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
    this.proxyTypes.put(cacheKey, proxy.getClass());
    return proxy;
  }

  this.advisedBeans.put(cacheKey, Boolean.FALSE);
  return bean;
}
```

## 1.找到advice、advisor

进入方法

```java
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
  //核心逻辑在这个方法
  List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
  if (advisors.isEmpty()) {
    return DO_NOT_PROXY;
  }
  return advisors.toArray();
}

protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
  //1.1.找到所有的Advisor
  List<Advisor> candidateAdvisors = findCandidateAdvisors();
  List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
  extendAdvisors(eligibleAdvisors);
  if (!eligibleAdvisors.isEmpty()) {
    eligibleAdvisors = sortAdvisors(eligibleAdvisors);
  }
  return eligibleAdvisors;
}
```

### 1.1.找到所有的Advisor

进入方法

```java
protected List<Advisor> findCandidateAdvisors() {
  //1.1.1.找到所有的Advisor
  List<Advisor> advisors = super.findCandidateAdvisors();
  //1.1.2.
  if (this.aspectJAdvisorsBuilder != null) {
    advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
  }
  return advisors;
}
```

#### 1.1.1.找到所有实现了Advisor接口的bean

具体逻辑实现在BeanFactoryAdvisorRetrievalHelper.findAdvisorBeans()方法中，大概的逻辑就是找到BeanFactory中实现了Advisor接口的bean

#### 1.1.2.找到@Aspect注解的bean

进入方法

```java
public List<Advisor> buildAspectJAdvisors() {
  List<String> aspectNames = this.aspectBeanNames;

  if (aspectNames == null) {
    synchronized (this) {
      aspectNames = this.aspectBeanNames;
      if (aspectNames == null) {
        List<Advisor> advisors = new ArrayList<>();
        aspectNames = new ArrayList<>();
        String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(this.beanFactory, Object.class, true, false);
        for (String beanName : beanNames) {
          if (!isEligibleBean(beanName)) {
            continue;
          }
          Class<?> beanType = this.beanFactory.getType(beanName);
          if (beanType == null) {
            continue;
          }
          //bean是否有@Aspect注解
          if (this.advisorFactory.isAspect(beanType)) {
            aspectNames.add(beanName);
            AspectMetadata amd = new AspectMetadata(beanType, beanName);
            if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
              MetadataAwareAspectInstanceFactory factory =
                new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
              //1.1.2.1.从aspect中找到所有的Advisor
              List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
              if (this.beanFactory.isSingleton(beanName)) {
                this.advisorsCache.put(beanName, classAdvisors);
              }
              else {
                this.aspectFactoryCache.put(beanName, factory);
              }
              advisors.addAll(classAdvisors);
            }
            else {
              // Per target or per this.
              if (this.beanFactory.isSingleton(beanName)) {
                throw new IllegalArgumentException("Bean with name '" + beanName +
                                                   "' is a singleton, but aspect instantiation model is not singleton");
              }
              MetadataAwareAspectInstanceFactory factory =
                new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
              this.aspectFactoryCache.put(beanName, factory);
              advisors.addAll(this.advisorFactory.getAdvisors(factory));
            }
          }
        }
        this.aspectBeanNames = aspectNames;
        return advisors;
      }
    }
  }

  if (aspectNames.isEmpty()) {
    return Collections.emptyList();
  }
  List<Advisor> advisors = new ArrayList<>();
  for (String aspectName : aspectNames) {
    List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
    if (cachedAdvisors != null) {
      advisors.addAll(cachedAdvisors);
    }
    else {
      MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
      advisors.addAll(this.advisorFactory.getAdvisors(factory));
    }
  }
  return advisors;
}
```

##### 1.1.2.1.从aspect中提取Advisor

进入方法

```java
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
  Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
  String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
  validate(aspectClass);

  // We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
  // so that it will only instantiate once.
  MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
    new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

  List<Advisor> advisors = new ArrayList<>();
  for (Method method : getAdvisorMethods(aspectClass)) {
    Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
    if (advisor != null) {
      advisors.add(advisor);
    }
  }

  // If it's a per target aspect, emit the dummy instantiating aspect.
  if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
    Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
    advisors.add(0, instantiationAdvisor);
  }

  // Find introduction fields.
  for (Field field : aspectClass.getDeclaredFields()) {
    Advisor advisor = getDeclareParentsAdvisor(field);
    if (advisor != null) {
      advisors.add(advisor);
    }
  }

  return advisors;
}
```



# 结语

在spring中，spring.factories文件是很重要的。所以虽然很简单，但肯定要知道

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。如有错误，请指正~)