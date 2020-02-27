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

  //@1.第一个核心子方法：找到advice和advisor
  Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
  if (specificInterceptors != DO_NOT_PROXY) {
    this.advisedBeans.put(cacheKey, Boolean.TRUE);
    //@2.第二个核心方法：创建代理
    Object proxy = createProxy(
      bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
    this.proxyTypes.put(cacheKey, proxy.getClass());
    return proxy;
  }

  this.advisedBeans.put(cacheKey, Boolean.FALSE);
  return bean;
}
```

## @1.找到advice和advisor

进入方法AbstractAdvisorAutoProxyCreator.getAdvicesAndAdvisorsForBean():

```java
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
  //核心逻辑在这个方法
  List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
  if (advisors.isEmpty()) {
    return DO_NOT_PROXY;
  }
  return advisors.toArray();
}
```

主要逻辑就是findEligibleAdvisors()方法，进入：

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
  //@1.1.找到所有的Advisor
  List<Advisor> candidateAdvisors = findCandidateAdvisors();
  //@1.2.找到当前的bean可以匹配的Advisor
  List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
  extendAdvisors(eligibleAdvisors);
  if (!eligibleAdvisors.isEmpty()) {
    eligibleAdvisors = sortAdvisors(eligibleAdvisors);
  }
  return eligibleAdvisors;
}
```

### @1.1.找到所有的Advisor

进入方法

```java
protected List<Advisor> findCandidateAdvisors() {
  //找到BeanFactory中直接实现了Advisor接口的bean
  List<Advisor> advisors = super.findCandidateAdvisors();
  if (this.aspectJAdvisorsBuilder != null) {
    //找到@Aspect注解的bean
    advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
  }
  return advisors;
}
```

具体逻辑实现在BeanFactoryAdvisorRetrievalHelper.findAdvisorBeans()方法中，大概的逻辑就是找到BeanFactory中实现了Advisor接口的bean

先是看有没有直接实现了Advisor接口的bean，然后就是找到@Aspect注解的bean。我们主要看@Aspect注解的bean转换为Advisor的过程：

进入方法this.aspectJAdvisorsBuilder.buildAspectJAdvisors()

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
        //遍历所有的bean
        for (String beanName : beanNames) {
          if (!isEligibleBean(beanName)) {
            continue;
          }
          Class<?> beanType = this.beanFactory.getType(beanName);
          //看看bean是否有@Aspect注解
          if (this.advisorFactory.isAspect(beanType)) {
            aspectNames.add(beanName);
            AspectMetadata amd = new AspectMetadata(beanType, beanName);
            if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
              MetadataAwareAspectInstanceFactory factory =
                new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
              //@1.1.1.从aspect中提取Advisor
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

#### 1.1.1.从aspect中提取Advisor

进入方法

```java
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
  //获取标记了@Aspect的类和beanName
  Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
  String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
  validate(aspectClass);
	
  MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
    new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

  List<Advisor> advisors = new ArrayList<>();
  //遍历没有标记@Pointcut的方法
  for (Method method : getAdvisorMethods(aspectClass)) {
    //@1.1.1.1.获取普通增强器
    Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
    if (advisor != null) {
      advisors.add(advisor);
    }
  }
  if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
    //对于延迟初始化的@Aspect注解的bean，生成SyntheticInstantiationAdvisor
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

##### @1.1.1.1.获取普通增强器

进入方法：

```java
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
                          int declarationOrderInAspect, String aspectName) {

  validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
	//@1.1.1.1.1.获取切点信息
  AspectJExpressionPointcut expressionPointcut = getPointcut(
    candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
  if (expressionPointcut == null) {
    return null;
  }
	//@1.1.1.1.2.根据切点信息生成Advisor
  return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
                                                        this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}
```

###### @1.1.1.1.1. 获取切点信息

```java
private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
  //找到所有的@Pointcut @Before @After @Around @AfterReturning @AfterThrowing注解
  AspectJAnnotation<?> aspectJAnnotation =
    AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
  if (aspectJAnnotation == null) {
    return null;
  }

  AspectJExpressionPointcut ajexp =
    new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
  //提取注解中的表达式
  ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
  if (this.beanFactory != null) {
    ajexp.setBeanFactory(this.beanFactory);
  }
  return ajexp;
}
```

###### @1.1.1.1.2.根据切点信息生成Advisor

可以看到，所有的增强都是由InstantiationModelAwarePointcutAdvisorImpl来统一封装的。进入方法：

```java
public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
                                                  Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
                                                  MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
	//属性赋值语句 省略
  ..

  if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
    // Static part of the pointcut is a lazy type.
    Pointcut preInstantiationPointcut = Pointcuts.union(
      aspectInstanceFactory.getAspectMetadata().getPerClausePointcut(), this.declaredPointcut);
    this.pointcut = new PerTargetInstantiationModelPointcut(
      this.declaredPointcut, preInstantiationPointcut, aspectInstanceFactory);
    this.lazy = true;
  } else {
    
    this.pointcut = this.declaredPointcut;
    this.lazy = false;
    //核心逻辑
    this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
  }
}
```

进入instantiateAdvice()方法：

```java
private Advice instantiateAdvice(AspectJExpressionPointcut pointcut) {
  Advice advice = this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pointcut,
                                                       this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
  return (advice != null ? advice : EMPTY_ADVICE);
}
```

继续深入到getAdvice()方法：

```java
public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
                        MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

  Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
  validate(candidateAspectClass);

  AspectJAnnotation<?> aspectJAnnotation =
    AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
  if (aspectJAnnotation == null) {
    return null;
  }

  AbstractAspectJAdvice springAdvice;
	//根据不同的注解生成不同的Advise
  switch (aspectJAnnotation.getAnnotationType()) {
    case AtPointcut:
      return null;
    case AtAround:
      //@Around
      springAdvice = new AspectJAroundAdvice(candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
      break;
    case AtBefore:
      springAdvice = new AspectJMethodBeforeAdvice(candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
      break;
    case AtAfter:
      springAdvice = new AspectJAfterAdvice(candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
      break;
    case AtAfterReturning:
      springAdvice = new AspectJAfterReturningAdvice(candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
      AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
      if (StringUtils.hasText(afterReturningAnnotation.returning())) {
        springAdvice.setReturningName(afterReturningAnnotation.returning());
      }
      break;
    case AtAfterThrowing:
      springAdvice = new AspectJAfterThrowingAdvice(candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
      AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
      if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
        springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
      }
      break;
    default:
      throw new UnsupportedOperationException(
        "Unsupported advice type on method: " + candidateAdviceMethod);
  }

  // Now to configure the advice...
  springAdvice.setAspectName(aspectName);
  springAdvice.setDeclarationOrder(declarationOrder);
  String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
  if (argNames != null) {
    springAdvice.setArgumentNamesFromStringArray(argNames);
  }
  springAdvice.calculateArgumentBindings();

  return springAdvice;
}
```

### @1.2.找到当前的bean可以匹配的Advisor

进入方法

```java
protected List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

  ProxyCreationContext.setCurrentProxiedBeanName(beanName);
  try {
    return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
  }
  finally {
    ProxyCreationContext.setCurrentProxiedBeanName(null);
  }
}
```

进入AopUtils.findAdvisorsThatCanApply()方法：

```java
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
  
  List<Advisor> eligibleAdvisors = new ArrayList<>();
  for (Advisor candidate : candidateAdvisors) {
    if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
      eligibleAdvisors.add(candidate);
    }
  }
  boolean hasIntroductions = !eligibleAdvisors.isEmpty();
  for (Advisor candidate : candidateAdvisors) {
    if (candidate instanceof IntroductionAdvisor) {
      // already processed
      continue;
    }
    //核心是这里的canApply方法
    if (canApply(candidate, clazz, hasIntroductions)) {
      eligibleAdvisors.add(candidate);
    }
  }
  return eligibleAdvisors;
}
```

进入AopUtils.canApply()方法：

```java
public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
	//先判断类是否匹配
  if (!pc.getClassFilter().matches(targetClass)) {
    return false;
  }
	//方法匹配器如果是MethodMatcher.TRUE，则返回匹配
  MethodMatcher methodMatcher = pc.getMethodMatcher();
  if (methodMatcher == MethodMatcher.TRUE) {
    return true;
  }

  IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
  if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
    introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
  }

  Set<Class<?>> classes = new LinkedHashSet<>();
  if (!Proxy.isProxyClass(targetClass)) {
    classes.add(ClassUtils.getUserClass(targetClass));
  }
  classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

  for (Class<?> clazz : classes) {
    Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
    //遍历目标方法
    for (Method method : methods) {
      if (introductionAwareMethodMatcher != null ?
          introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
          methodMatcher.matches(method, targetClass)) {
        return true;
      }
    }
  }

  return false;
}
```

## @2.创建代理

进入方法：

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
                             @Nullable Object[] specificInterceptors, TargetSource targetSource) {

  if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
    AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
  }

  ProxyFactory proxyFactory = new ProxyFactory();
  proxyFactory.copyFrom(this);

  if (!proxyFactory.isProxyTargetClass()) {
    if (shouldProxyTargetClass(beanClass, beanName)) {
      proxyFactory.setProxyTargetClass(true);
    }
    else {
      evaluateProxyInterfaces(beanClass, proxyFactory);
    }
  }

  Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
  proxyFactory.addAdvisors(advisors);
  proxyFactory.setTargetSource(targetSource);
  customizeProxyFactory(proxyFactory);

  proxyFactory.setFrozen(this.freezeProxy);
  if (advisorsPreFiltered()) {
    proxyFactory.setPreFiltered(true);
  }

  return proxyFactory.getProxy(getProxyClassLoader());
}
```



# 结语

在spring中，spring.factories文件是很重要的。所以虽然很简单，但肯定要知道

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。如有错误，请指正~)