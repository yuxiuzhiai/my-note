# 概念

当spring解析完配置文件后，配置文件只是转换成了BeanDefinition，并不是具体的bean。如果想要得到一个具体的bean，还需要经过bean的实例化过程。

# 用法

还是借助分析xml文件加载时候的测试方法：

```java
@Test
public void test(){
  //创建一个实现了BeanDefinitionRegistry的BeanFactory实现
  //DefaultListableBeanFactory也是绝大多数场景下，BeanFactory的具体实现
  DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
  //XmlBeanDefinitionReader创建，从名字可以看出来 这个类是用来从xml文件中读取配置的
  XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanFactory);
  //具体解析xml文件中的配置，并注册到BeanDefinitionRegistry中去
  reader.loadBeanDefinitions(new ClassPathResource("xmlBeanDefinition.xml"));

  UserInfo userInfo = beanFactory.getBean(UserInfo.class);
  System.out.println(userInfo.getA());
}
```

# 实现

主要的逻辑在AbstractBeanFactory.doGetBean方法中：

```java
protected <T> T doGetBean(final String name, final Class<T> requiredType, Object[] args, boolean typeCheckOnly){
  //@1.转换beanName
  final String beanName = transformedBeanName(name);
  Object bean;
  //@2.尝试从单例缓存中加载
  Object sharedInstance = getSingleton(beanName);
  if (sharedInstance != null && args == null) {
    //见下
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  }else {
    if (isPrototypeCurrentlyInCreation(beanName)) {
      throw new BeanCurrentlyInCreationException(beanName);
    }
    BeanFactory parentBeanFactory = getParentBeanFactory();
    //如果parentBeanFactory不为空，并且当前BeanFactory不包含这个beanName，则递归地从父BeanFactory尝试创建bean
    if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
      ..
    }
    if (!typeCheckOnly) {
      markBeanAsCreated(beanName);
    }
    //转换为RootBeanDefinition，就是看这个bean是不是一个子bean，如果是的话，就先merge父bean的属性
    final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
    checkMergedBeanDefinition(mbd, beanName, args);
    //看看bean有没有依赖，如果有的话，就先创建依赖
    String[] dependsOn = mbd.getDependsOn();
    if (dependsOn != null) {
      for (String dep : dependsOn) {
        if (isDependent(beanName, dep)) {
          throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                          "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
        }
        registerDependentBean(dep, beanName);
        getBean(dep);
      }
    }
    // Create bean instance.
    if (mbd.isSingleton()) {
      sharedInstance = getSingleton(beanName, () -> {
        //@3.创建bean
        return createBean(beanName, mbd, args);
      });
      //7
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    }

    else if (mbd.isPrototype()) {
      // It's a prototype -> create a new instance.
      Object prototypeInstance = null;
      try {
        beforePrototypeCreation(beanName);
        prototypeInstance = createBean(beanName, mbd, args);
      }
      finally {
        afterPrototypeCreation(beanName);
      }
      bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
    }
    else {
      String scopeName = mbd.getScope();
      final Scope scope = this.scopes.get(scopeName);
      Object scopedInstance = scope.get(beanName, () -> {
        beforePrototypeCreation(beanName);
        try {
          return createBean(beanName, mbd, args);
        }
        finally {
          afterPrototypeCreation(beanName);
        }
      });
      bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
    }
  }
}

// Check if required type matches the type of the actual bean instance.
if (requiredType != null && !requiredType.isInstance(bean)) {
  T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
  if (convertedBean == null) {
    throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
  }
  return convertedBean;
}
}
return (T) bean;
}
```

## @1.转换beanName

这里的beanName有三种可能：

* 就是bean的name或者说是id，那不做处理
* &开头的FactoryBean，则把&去掉
* 不是一个beanName，是一个alias，则先通过AliasRegistry找到对应的beanName然后返回

## @2.尝试从单例缓存中加载

2对应的方法DefaultSingletonBeanRegistry.getSingleton()

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  //先检查已经创建好的bean里面，有没有这个beanName的
  Object singletonObject = this.singletonObjects.get(beanName);
  //如果bean没有创建好，并且正在创建
  if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
    synchronized (this.singletonObjects) {
      singletonObject = this.earlySingletonObjects.get(beanName);
      if (singletonObject == null && allowEarlyReference) {
        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
        if (singletonFactory != null) {
          singletonObject = singletonFactory.getObject();
          this.earlySingletonObjects.put(beanName, singletonObject);
          this.singletonFactories.remove(beanName);
        }
      }
    }
  }
  return singletonObject;
}
```

先介绍下DefaultSingletonBeanRegistry中的各个map/set的作用：

* singletonObjects:保存了beanName->bean实例的map
* singletonFactories：保存了beanName->ObjectFactory的map
* earlySingletonObjects：也是beanName->bean实例，但是是在bean还没创建完的时候就加进去，用来检测循环引用的
* registeredSingletons：Set，保存已经注册的bean
* singletonsCurrentlyInCreation：Set，保存正在创建过程中的bean

## 3.从bean的实例中获取对象

```java

```



## @3.创建bean

作用就是根据一个RootBeanDefinition去把这个bean真的创建出来

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
  RootBeanDefinition mbdToUse = mbd;
  //确定beanClass
  Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
  if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
    mbdToUse = new RootBeanDefinition(mbd);
    mbdToUse.setBeanClass(resolvedClass);
  }
  //准备lookup-method和replace-method的处理
  mbdToUse.prepareMethodOverrides();
  //@3.1.给BeanPostProcessor一个机会来返回代理替代真正的实例
  Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
  if (bean != null) {
    return bean;
  }
  //@3.2执行bean的创建
  Object beanInstance = doCreateBean(beanName, mbdToUse, args);
  return beanInstance;
}
```

### @3.1.BeanPostProcessor处理

```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
  Object bean = null;
  if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
    // Make sure bean class is actually resolved at this point.
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      Class<?> targetType = determineTargetType(beanName, mbd);
      if (targetType != null) {
        //@3.1.1.实例化前的BeanPostProcessor应用
        bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
        if (bean != null) {
          //@3.1.2.实例化后的BeanPostProcessor应用.是要做bean不为空的前提下，才会在这里调用的
          bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
        }
      }
    }
    mbd.beforeInstantiationResolved = (bean != null);
  }
  return bean;
}
```

#### @3.1.1.实例化前的BeanPostProcessor应用

```java
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
  for (BeanPostProcessor bp : getBeanPostProcessors()) {
    if (bp instanceof InstantiationAwareBeanPostProcessor) {
      InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
      Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
      if (result != null) {
        return result;
      }
    }
  }
  return null;
}
```

### @3.1.2.实例化后的BeanPostProcessor应用

```java
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) {
  Object result = existingBean;
  for (BeanPostProcessor processor : getBeanPostProcessors()) {
    Object current = processor.postProcessAfterInitialization(result, beanName);
    if (current == null) {
      return result;
    }
    result = current;
  }
  return result;
}
```



### @3.2.执行bean的创建

进入方法doCreateBean

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, Object[] args) {
  BeanWrapper instanceWrapper = null;
  if (mbd.isSingleton()) {
    instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
  }
  //@3.2.1.创建一个BeanWrapper
  if (instanceWrapper == null) {
    instanceWrapper = createBeanInstance(beanName, mbd, args);
  }
  final Object bean = instanceWrapper.getWrappedInstance();
  Class<?> beanType = instanceWrapper.getWrappedClass();
  if (beanType != NullBean.class) {
    mbd.resolvedTargetType = beanType;
  }

  //MergedBeanDefinitionPostProcessor的应用
  synchronized (mbd.postProcessingLock) {
    if (!mbd.postProcessed) {
      applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
      mbd.postProcessed = true;
    }
  }
  // Eagerly cache singletons to be able to resolve circular references
  // even when triggered by lifecycle interfaces like BeanFactoryAware.
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                    isSingletonCurrentlyInCreation(beanName));
  if (earlySingletonExposure) {
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  }
  // Initialize the bean instance.
  Object exposedObject = bean;
  //@3.2.2.属性填充
  populateBean(beanName, mbd, instanceWrapper);
  //@3.2.3.初始化bean
  exposedObject = initializeBean(beanName, exposedObject, mbd);
  if (earlySingletonExposure) {
    Object earlySingletonReference = getSingleton(beanName, false);
    if (earlySingletonReference != null) {
      if (exposedObject == bean) {
        exposedObject = earlySingletonReference;
      }
      else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
        String[] dependentBeans = getDependentBeans(beanName);
        Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
        for (String dependentBean : dependentBeans) {
          if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
            actualDependentBeans.add(dependentBean);
          }
        }
      }
    }
  }
  // Register bean as disposable.
  registerDisposableBeanIfNecessary(beanName, bean, mbd);
  return exposedObject;
}
```

#### @3.2.1.创建一个BeanWrapper出来

进入AbstractAutowireCapableBeanFactory.createBeanInstance方法

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.
		Class<?> beanClass = resolveBeanClass(mbd, beanName);
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}
		//看看是否有instanceSupplier属性，如果有就用这个Supplier来创建BeanWrapper
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}
		//
		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				return instantiateBean(beanName, mbd);
			}
		}

		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
      //找到一个合适的有参数的构造函数
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
      //3.1.1.1.找到一个合适的有参数的构造函数TODO
			return autowireConstructor(beanName, mbd, ctors, null);
		}
		//3.1.1.2.没有啥特殊处理，直接创建TODO
		return instantiateBean(beanName, mbd);
	}
```

#### @3.2.2.属性填充

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
	//InstantiationAwareBeanPostProcessor的应用
  if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
      if (bp instanceof InstantiationAwareBeanPostProcessor) {
        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
        if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
          return;
        }
      }
    }
  }

  PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
	//注入类型 byName或者是byType
  int resolvedAutowireMode = mbd.getResolvedAutowireMode();
  if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
    MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
    //@3.2.2.1.根据name填充属性
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
      autowireByName(beanName, mbd, bw, newPvs);
    }
    //@3.2.2.2.根据type填充属性
    if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
      autowireByType(beanName, mbd, bw, newPvs);
    }
    pvs = newPvs;
  }

  boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
  boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

  PropertyDescriptor[] filteredPds = null;
  if (hasInstAwareBpps) {
    if (pvs == null) {
      pvs = mbd.getPropertyValues();
    }
    //InstantiationAwareBeanPostProcessor的应用
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
      if (bp instanceof InstantiationAwareBeanPostProcessor) {
        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
        PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
        if (pvsToUse == null) {
          if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
          }
          pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
          if (pvsToUse == null) {
            return;
          }
        }
        pvs = pvsToUse;
      }
    }
  }
  if (needsDepCheck) {
    if (filteredPds == null) {
      filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
    }
    checkDependencies(beanName, mbd, filteredPds, pvs);
  }

  if (pvs != null) {
    //将属性注入到BeanWrapper中
    applyPropertyValues(beanName, mbd, bw, pvs);
  }
}
```

##### @3.2.2.1.根据name填充属性

```java
protected void autowireByName(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
	//获取需要依赖注入的属性
  String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
  for (String propertyName : propertyNames) {
    if (containsBean(propertyName)) {
      //递归地初始化相关的bean
      Object bean = getBean(propertyName);
      pvs.add(propertyName, bean);
      registerDependentBean(propertyName, beanName);
    }
  }
}
```

##### @3.2.2.2.根据type填充属性

```java
protected void autowireByType(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

  TypeConverter converter = getCustomTypeConverter();
  if (converter == null) {
    converter = bw;
  }

  Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
  //获取需要依赖注入的属性
  String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
  for (String propertyName : propertyNames) {
      PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
      if (Object.class != pd.getPropertyType()) {
        MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
        // Do not allow eager init for type matching in case of a prioritized post-processor.
        boolean eager = !(bw.getWrappedInstance() instanceof PriorityOrdered);
        DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
        Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
        if (autowiredArgument != null) {
          pvs.add(propertyName, autowiredArgument);
        }
        for (String autowiredBeanName : autowiredBeanNames) {
          registerDependentBean(autowiredBeanName, beanName);
        }
        autowiredBeanNames.clear();
      }
  }
}
```

#### @3.2.3.初始化bean

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
	//对于实现了各种*Aware的bean，注入对应的类
  invokeAwareMethods(beanName, bean);
  Object wrappedBean = bean;
  if (mbd == null || !mbd.isSynthetic()) {
    //BeanPostProcessor的应用
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
  }
	//如果bean实现了InitializingBean接口或者自定义了init-method，会在这里执行
  invokeInitMethods(beanName, wrappedBean, mbd);
  if (mbd == null || !mbd.isSynthetic()) {
    //BeanPostProcessor的应用
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
  }
  return wrappedBean;
}
```

# 结语

bean的创建基本就是spring中最为复杂的过程了。代码逻辑非常复杂，涉及到的各种类和方法也非常多

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。如有错误，请指正~)