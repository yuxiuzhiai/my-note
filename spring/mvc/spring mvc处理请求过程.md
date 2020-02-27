# 概念

本文会介绍一个请求到达spring mvc的整个处理过程。

我们知道在spring mvc中，一般是会有DispatcherServlet统一处理所有的请求。接下来一起看看整个的流程。

先看下核心类DispatcherServlet的类图

<img src="/Users/didi/workspace/study/my-note/pic/DispatcherServlet.png" style="zoom:50%;" />

所以DispatcherServlet首先是一个普通的Servlet实现，其次，它实现了ApplicationContextAware，所以可以在运行时获取到ApplicationContext，结合到spring的上下文中。所以DIspatcherServlet是Servlet跟spring相互结合的一个纽带。

# DispatcherServlet的初始化

当第一个请求到来时，Servlet需要首先执行init初始化。进入父类方法：HttpServletBean.init()

```java
public final void init() throws ServletException {
  //解析init-param，并封装到PropertyValues
  PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
  if (!pvs.isEmpty()) {
    BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
    ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
    //注册Resource类型的PropertyEditor
    bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
    initBeanWrapper(bw);
    bw.setPropertyValues(pvs, true);
  }

  //@1.留给子类用于初始化
  initServletBean();
}
```

## @1.FrameworkServlet.initServletBean()方法

```java
protected final void initServletBean() throws ServletException {
	//@1.1.WebApplicationContext的初始化
  this.webApplicationContext = initWebApplicationContext();
  //空方法
  initFrameworkServlet();
}
```

### @1.1.WebApplicationContext的初始化

```java
protected WebApplicationContext initWebApplicationContext() {
  //获取ApplicationContext
  WebApplicationContext rootContext = WebApplicationContextUtils.getWebApplicationContext(getServletContext());
  WebApplicationContext wac = null;
  //如果rootContext不为空，则直接使用；如果为空则找ServletContext中是否设置了；如果还是找不到就自己创建一个
  if (this.webApplicationContext != null) {
    wac = this.webApplicationContext;
    if (wac instanceof ConfigurableWebApplicationContext) {
      ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
      if (!cwac.isActive()) {
        if (cwac.getParent() == null) {
          cwac.setParent(rootContext);
        }
        configureAndRefreshWebApplicationContext(cwac);
      }
    }
  }
  if (wac == null) {
    wac = findWebApplicationContext();
  }
  if (wac == null) {
    wac = createWebApplicationContext(rootContext);
  }

  if (!this.refreshEventReceived) {
    synchronized (this.onRefreshMonitor) {
      //@1.1.1.刷新
      onRefresh(wac);
    }
  }

  if (this.publishContext) {
    //将WebApplicationContext作为一个属性设置到ServletContext中去
    String attrName = getServletContextAttributeName();
    getServletContext().setAttribute(attrName, wac);
  }

  return wac;
}
```

#### @1.1.1.刷新

```java
//主要逻辑在DispatcherServlet的这个方法中
protected void initStrategies(ApplicationContext context) {
  //构建文件上传相关的bean
  initMultipartResolver(context);//@1
  //构建locale相关的bean
  initLocaleResolver(context);
  //构建theme相关的bean
  initThemeResolver(context);
  //构建HandlerMapping，用于将请求映射到handler上
  initHandlerMappings(context);
  //构建HandlerAdapter，执行handler相关
  initHandlerAdapters(context);
  //构建HandlerExceptionResolver，处理异常相关
  initHandlerExceptionResolvers(context);
  initRequestToViewNameTranslator(context);
  initViewResolvers(context);
  //构建FlashMapManager
  initFlashMapManager(context);
}
```

* 文件上传相关，请看https://blog.csdn.net/yuxiuzhiai/article/details/103483795
* locale相关，请看https://blog.csdn.net/yuxiuzhiai/article/details/103517935
* theme相关，请看https://blog.csdn.net/yuxiuzhiai/article/details/103517954
* HandlerMapping相关，请看https://blog.csdn.net/yuxiuzhiai/article/details/103501621
* 处理器、HandlerAdapter相关，请看https://blog.csdn.net/yuxiuzhiai/article/details/103501568
* 异常处理HandlerExceptionResolver相关，请看https://blog.csdn.net/yuxiuzhiai/article/details/103518358
* RequestToViewNameTranslator这个很简单不用单独说，就是根据请求找到一个view的name
* ViewResolver相关，请看https://blog.csdn.net/yuxiuzhiai/article/details/103518390
* FlashMap相关，请看https://blog.csdn.net/yuxiuzhiai/article/details/103518404

# 请求处理

在FrameworkServlet中，我们发现，无论是get put post delete方法，都会最终转发到FrameworkServlet的processRequest()方法中：

```java
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
  throws ServletException, IOException {

  long startTime = System.currentTimeMillis();
  Throwable failureCause = null;
	//根据request创建LocaleContext和RequestAttributes，并绑定到线程的ThreadLocal上
  LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
  LocaleContext localeContext = buildLocaleContext(request);

  RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
  ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);
	
  WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
  asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());

  initContextHolders(request, localeContext, requestAttributes);

  try {
    //核心逻辑
    doService(request, response);
  } finally {
    resetContextHolders(request, previousLocaleContext, previousAttributes);
    if (requestAttributes != null) {
      requestAttributes.requestCompleted();
    }
    //输出日志
    logResult(request, response, failureCause, asyncManager);
    //发布请求处理结束事件
    publishRequestHandledEvent(request, response, startTime, failureCause);
  }
}
```

进入doService()方法：

```java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
  logRequest(request);

  // Keep a snapshot of the request attributes in case of an include,
  // to be able to restore the original attributes after the include.
  Map<String, Object> attributesSnapshot = null;
  if (WebUtils.isIncludeRequest(request)) {
    attributesSnapshot = new HashMap<>();
    Enumeration<?> attrNames = request.getAttributeNames();
    while (attrNames.hasMoreElements()) {
      String attrName = (String) attrNames.nextElement();
      if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
        attributesSnapshot.put(attrName, request.getAttribute(attrName));
      }
    }
  }

  //给request对象设置上各种属性
  request.setAttribute(..,..);
  ...

  try {
    //好了，其实到这里才是真正的逻辑处理
    doDispatch(request, response);
  }
  finally {
    if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
      // Restore the original attribute snapshot, in case of an include.
      if (attributesSnapshot != null) {
        restoreAttributesAfterInclude(request, attributesSnapshot);
      }
    }
  }
}
```

好了，终于找到了核心逻辑：DispatcherServlet的dispatch()方法

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
  HttpServletRequest processedRequest = request;
  HandlerExecutionChain mappedHandler = null;
  boolean multipartRequestParsed = false;

  WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

  try {
    ModelAndView mv = null;
    Exception dispatchException = null;
    try {
      //check是否是文件上传请求:就是判断Content-Type是否是multipart开头的，如果是，就把request处理成MultipartRequest
      processedRequest = checkMultipart(request);
      multipartRequestParsed = (processedRequest != request);

      //@1.找到对应的handler
      mappedHandler = getHandler(processedRequest);
      if (mappedHandler == null) {
        //@2.没找到handler的处理
        noHandlerFound(processedRequest, response);
        return;
      }

      //根据handler的类型找到一个对应的HandlerAdapter
      HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

      //如果handler支持的话，处理last-modified头部
      String method = request.getMethod();
      boolean isGet = "GET".equals(method);
      if (isGet || "HEAD".equals(method)) {
        long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
        if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
          return;
        }
      }
			//执行HandlerInterceptor的preHandler方法
      if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        return;
      }

      //@3.执行handler
      mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

      if (asyncManager.isConcurrentHandlingStarted()) {
        return;
      }

      applyDefaultViewName(processedRequest, mv);
      mappedHandler.applyPostHandle(processedRequest, response, mv);
    }
    catch (Exception ex) {
      dispatchException = ex;
    }
    catch (Throwable err) {
      // As of 4.3, we're processing Errors thrown from handler methods as well,
      // making them available for @ExceptionHandler methods and other scenarios.
      dispatchException = new NestedServletException("Handler dispatch failed", err);
    }
    processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
  }
  catch (Exception ex) {
    triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
  }
  catch (Throwable err) {
    triggerAfterCompletion(processedRequest, response, mappedHandler,
                           new NestedServletException("Handler processing failed", err));
  }
  finally {
    if (asyncManager.isConcurrentHandlingStarted()) {
      // Instead of postHandle and afterCompletion
      if (mappedHandler != null) {
        mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
      }
    }
    else {
      // Clean up any resources used by a multipart request.
      if (multipartRequestParsed) {
        cleanupMultipart(processedRequest);
      }
    }
  }
}
```

## @1.找到对应的handler

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
  if (this.handlerMappings != null) {
    for (HandlerMapping mapping : this.handlerMappings) {
      HandlerExecutionChain handler = mapping.getHandler(request);
      if (handler != null) {
        return handler;
      }
    }
  }
  return null;
}
```

进入

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
	//cors的处理
  if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
    CorsConfiguration config = (this.corsConfigurationSource != null ? this.corsConfigurationSource.getCorsConfiguration(request) : null);
    CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
    config = (config != null ? config.combine(handlerConfig) : handlerConfig);
    executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
  }
  return executionChain;
}
```

## @2.没找到handler的处理

```java
protected void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
  if (this.throwExceptionIfNoHandlerFound) {
    //如果配置了throwExceptionIfNoHandlerFound属性，则抛出异常
    throw new NoHandlerFoundException(request.getMethod(), getRequestUri(request),
                                      new ServletServerHttpRequest(request).getHeaders());
  } else {
    //否则设置返回404
    response.sendError(HttpServletResponse.SC_NOT_FOUND);
  }
}
```

## @3.执行handler

由于有不同种类的handler，所以我们已最复杂的RequestMappingHandlerAdapter为例。主要的逻辑在RequestMappingHandlerAdapter.invokeHandlerMethod()方法

```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
                                           HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

  ServletWebRequest webRequest = new ServletWebRequest(request, response);
  try {
    WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
    ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

    ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
    if (this.argumentResolvers != null) {
      invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
    }
    if (this.returnValueHandlers != null) {
      invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
    }
    invocableMethod.setDataBinderFactory(binderFactory);
    invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

    ModelAndViewContainer mavContainer = new ModelAndViewContainer();
    mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
    modelFactory.initModel(webRequest, mavContainer, invocableMethod);
    mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

    AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
    asyncWebRequest.setTimeout(this.asyncRequestTimeout);

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
    asyncManager.setTaskExecutor(this.taskExecutor);
    asyncManager.setAsyncWebRequest(asyncWebRequest);
    asyncManager.registerCallableInterceptors(this.callableInterceptors);
    asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

    if (asyncManager.hasConcurrentResult()) {
      Object result = asyncManager.getConcurrentResult();
      mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
      asyncManager.clearConcurrentResult();
      LogFormatUtils.traceDebug(logger, traceOn -> {
        String formatted = LogFormatUtils.formatValue(result, !traceOn);
        return "Resume with async result [" + formatted + "]";
      });
      invocableMethod = invocableMethod.wrapConcurrentResult(result);
    }

    invocableMethod.invokeAndHandle(webRequest, mavContainer);
    if (asyncManager.isConcurrentHandlingStarted()) {
      return null;
    }

    return getModelAndView(mavContainer, modelFactory, webRequest);
  }
  finally {
    webRequest.requestCompleted();
  }
}
```





## 结语

在spring中，spring.factories文件是很重要的。所以虽然很简单，但肯定要知道

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。
如有错误，请指正~)