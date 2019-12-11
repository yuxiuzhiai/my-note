## 概念

根据请求找到对应的处理器Handler。另外，Handler总是会包装成一个HandlerExecutionChain实例,该实例可能会有一些HandlerInterceptor实例.DispatcherServlet总是会调用每个HandlerInterceptor的preHandle方法,如果所有的preHandle方法都返回true才会调用handler本身

## 实现

### 接口定义

```java
public interface HandlerMapping {

	String BEST_MATCHING_HANDLER_ATTRIBUTE = HandlerMapping.class.getName() + ".bestMatchingHandler";

	String LOOKUP_PATH = HandlerMapping.class.getName() + ".lookupPath";

	String PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE = HandlerMapping.class.getName() + ".pathWithinHandlerMapping";

	String BEST_MATCHING_PATTERN_ATTRIBUTE = HandlerMapping.class.getName() + ".bestMatchingPattern";

	String INTROSPECT_TYPE_LEVEL_MAPPING = HandlerMapping.class.getName() + ".introspectTypeLevelMapping";

	String URI_TEMPLATE_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".uriTemplateVariables";

	String MATRIX_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".matrixVariables";

	String PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE = HandlerMapping.class.getName() + ".producibleMediaTypes";
	//根据请求找到对应的Handler
	HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
```

### 具体实现

在DispatcherServlet中，并不是只有一个HandlerMapping，而是有个HandlerMapping的列表，每当有请求过来，DispatcherServlet会依次遍历HandlerMapping，直到找到对应的Handler为止。

从实现来说，HandlerMapping的实现是一种典型的模板方法模式。

在公共的抽象类中定义了getHandler()的实现

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		Object handler = getHandlerInternal(request);
		if (handler == null) {
			handler = getDefaultHandler();
		}
		if (handler == null) {
			return null;
		}
		// Bean name or resolved handler?
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = obtainApplicationContext().getBean(handlerName);
		}

		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

		if (logger.isTraceEnabled()) {
			logger.trace("Mapped to " + handler);
		}
		else if (logger.isDebugEnabled() && !request.getDispatcherType().equals(DispatcherType.ASYNC)) {
			logger.debug("Mapped to " + executionChain.getHandler());
		}

		if (hasCorsConfigurationSource(handler)) {
			CorsConfiguration config = (this.corsConfigurationSource != null ? this.corsConfigurationSource.getCorsConfiguration(request) : null);
			CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
			config = (config != null ? config.combine(handlerConfig) : handlerConfig);
			executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
		}

		return executionChain;
	}
```

而每个具体的实现类重写getDefaultHandler()方法。

### 子类

类图如下

<img src="/Users/didi/workspace/study/my-note/pic/RequestMappingHandlerMapping.png" alt="RequestMappingHandlerMapping" style="zoom:80%;" />

从使用的角度来看，我们要将一个请求映射到一个处理器，有这几个办法：

* 通过@RequestMapping注解。这种方式，对应的HandlerMapping就是RequestMappingHandlerMapping
* 通过定义一个名字以/开头的bean。这个比较少见，即如果bean的name是/开头，比如/foo，那请求的path如果是/foo那就会交由这个bean来处理。对应的HandlerMapping是BeanNameUrlHandlerMapping
* SimpleUrlHandlerMapping跟BeanNameUrlHandlerMapping差不多，可以自己手动将一个path映射到一个bean上
* WelcomPageHandlerMapping就是映射到index.html

### @RequestMapping的实现

我们知道，@Component是将一个普通的Java类注册成一个Spring的bean。@Controller是一个特殊的@Component。特殊性就在于HandlerMapping对于@Controller/@RestController/@RequestMapping有特殊的处理。

AbstractHandlerMethodMapping实现了InitializingBean，所以有这么个方法：

```java
@Override
public void afterPropertiesSet() {
  initHandlerMethods();//@1
}
//@1
protected void initHandlerMethods() {
  //getCandidateBeanNames()基本上就是获取所有ApplicationContext中的bean的name
  for (String beanName : getCandidateBeanNames()) {
    //只有name不是以scopedTarget开头就执行@2
    if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
      processCandidateBean(beanName);//@2
    }
  }
  handlerMethodsInitialized(getHandlerMethods());
}
//@2
protected void processCandidateBean(String beanName) {
  Class<?> beanType = null;
  try {
    beanType = obtainApplicationContext().getType(beanName);
  }
  catch (Throwable ex) {
    // An unresolvable bean type, probably from a lazy bean - let's ignore it.
    if (logger.isTraceEnabled()) {
      logger.trace("Could not resolve type for bean '" + beanName + "'", ex);
    }
  }
  //判断是不是一个handler
  if (beanType != null && isHandler(beanType)) {//@3
    detectHandlerMethods(beanName);
  }
}
//@3
protected boolean isHandler(Class<?> beanType) {
  //这个bean是不是有@Controller注解或者@RequestMapping注解
  return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
          AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
}
```

上述的逻辑很简单粗暴，就是获取所有bean，看看bean的类上是不是有@Controller或者@RequestMapping注解。如果有，再去解析内部的方法。

解析内部方法的过程：

```java
protected void detectHandlerMethods(Object handler) {
  Class<?> handlerType = (handler instanceof String ?
                          obtainApplicationContext().getType((String) handler) : handler.getClass());

  if (handlerType != null) {
    Class<?> userType = ClassUtils.getUserClass(handlerType);
   	//获取的是Method->RequestMappingInfo的Map
    Map<Method, T> methods = MethodIntrospector.selectMethods(userType,(MethodIntrospector.MetadataLookup<T>) method -> {
      try {
        return getMappingForMethod(method, userType);//@4
      }
      catch (Throwable ex) {
        throw new IllegalStateException("Invalid mapping on handler class [" +
                                        userType.getName() + "]: " + method, ex);
      }
    });
    if (logger.isTraceEnabled()) {
      logger.trace(formatMappings(userType, methods));
    }
    methods.forEach((method, mapping) -> {
      Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
      //将这个方法注册到url->handler的Map中去
      registerHandlerMethod(handler, invocableMethod, mapping);//@6
    });
  }
}
```

先看看@4的实现

```java
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
  RequestMappingInfo info = createRequestMappingInfo(method);//@5
  if (info != null) {
    //这里还有个createRequestMappingInfo是因为@RequestMapping即可以注解在类上，也可以注解在方法上，所以会有个先方法的，然后再用类的其他值补全
    RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
    if (typeInfo != null) {
      info = typeInfo.combine(info);
    }
    String prefix = getPathPrefix(handlerType);
    if (prefix != null) {
      info = RequestMappingInfo.paths(prefix).build().combine(info);
    }
  }
  return info;
}
//@5
private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
  //获取方法上的@RequestMapping注解
  RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
  RequestCondition<?> condition = (element instanceof Class ?
                                   getCustomTypeCondition((Class<?>) element) : getCustomMethodCondition((Method) element));
  //根据注解的各个值创建一个RequestMappingInfo，这个RequestMappingInfo其实就是组合了@RequestMapping注解的各个字段的值，method啊，header啊，consumer啊之类的
  return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
}
```

再继续看看注册这个映射@6的实现的实现

```java
//mapping是RequestMappingInfo，handler是@Controller注解的bean name，method是@RequestMapping注解的方法
public void register(T mapping, Object handler, Method method) {
  //将这个method和一些关联的信息组合成handler
  HandlerMethod handlerMethod = createHandlerMethod(handler, method);//@7
  //验证这个RequestMappingInfo是不是已经有对应的handler了，有的话就报异常
  validateMethodMapping(handlerMethod, mapping);
  this.mappingLookup.put(mapping, handlerMethod);
	//获取到关联的url，可以有多个
  List<String> directUrls = getDirectUrls(mapping);
  for (String url : directUrls) {
    this.urlLookup.add(url, mapping);
  }

  String name = null;
  //生成一个名字，比如这个类是DemoController，方法是index，那生成的名字就是DC#index
  if (getNamingStrategy() != null) {
    name = getNamingStrategy().getName(handlerMethod, mapping);
    addMappingName(name, handlerMethod);
  }
	//cors的配置，详情见https://blog.csdn.net/yuxiuzhiai/article/details/103501443
  CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
  if (corsConfig != null) {
    this.corsLookup.put(handlerMethod, corsConfig);
  }

  this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));
}

//@7
protected HandlerMethod createHandlerMethod(Object handler, Method method) {
  if (handler instanceof String) {
    //这个HandlerMethod就是把这个方法，包含这个方法的beanName，已经整个BeanFactory，方法的参数这些组装到一起，形成一个hanler
    return new HandlerMethod((String) handler,
                             obtainApplicationContext().getAutowireCapableBeanFactory(), method);
  }
  return new HandlerMethod(handler, method);
}
```

### 梳理一遍

* AbstractHandlerMethodMapping实现了InitializingBean，所以会触发initHandlerMethods()方法
* initHandlerMethods()就是粗暴的遍历所有bean，看他是不是满足成为一个handler的条件，条件就是@Controller和@RequestMapping主机
* 如果有类和方法满足，则根绝@RequestMapping注解，生成一个RequestMappingInfo，然后根据这个RequestMappingInfo和这个类、这个方法、方法参数这些构造出一个HandlerMethod
* 保存这个url到这个HandlerMethod的映射
* 处理下cors
* 结束

### next

到这还没结束，上述的所有只是说了spring是怎么样将@Controller和@RequestMapping注解的方法和类整成一个可以处理请求的handler的。而spring中并不是用这个裸handler处理请求的。接着看最开始的方法(又回到最初的梦想)

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		//我们知道，如果是RequestMappingHandlerMapping处理的，那这里返回的是一个HandlerMethod
  	Object handler = getHandlerInternal(request);
		if (handler == null) {
			handler = getDefaultHandler();
		}
		if (handler == null) {
			return null;
		}
		// Bean name or resolved handler?
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = obtainApplicationContext().getBean(handlerName);
		}
		//关于handler的东西请见https://blog.csdn.net/yuxiuzhiai/article/details/103501568
		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

		if (logger.isTraceEnabled()) {
			logger.trace("Mapped to " + handler);
		}
		else if (logger.isDebugEnabled() && !request.getDispatcherType().equals(DispatcherType.ASYNC)) {
			logger.debug("Mapped to " + executionChain.getHandler());
		}
		//cors处理
		if (hasCorsConfigurationSource(handler)) {
			CorsConfiguration config = (this.corsConfigurationSource != null ? this.corsConfigurationSource.getCorsConfiguration(request) : null);
			CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
			config = (config != null ? config.combine(handlerConfig) : handlerConfig);
			executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
		}
		//返回包裹了拦截器链的handler
		return executionChain;
	}
```

关于handler和mvc 方法拦截器的解析请见https://blog.csdn.net/yuxiuzhiai/article/details/103501568

关于spring mvc的cors的处理请见https://blog.csdn.net/yuxiuzhiai/article/details/103501443

## 结语

整个HandlerMapping的作用就是根据请求返回一个包裹这拦截器链的handler。

总共涉及25个类(HandlerMapping12，RequestCondition11，HandlerMethodMappingNamingStrategy2)

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。
如有错误，请指正~)