## 概念

我们知道在spring mvc中，当来了个请求的时候，首先要根据HandlerMapping去找到handler，然后就是要去执行这个handler，但是handler的种类不同，自然也需要不同的执行方式。这里的HandlerAdapter就可以当成是handler的执行方式，负责具体地去执行这个handler

## 用法

handler个几种类型

* HandlerMethod：基于@Controller和@RequestMapping注解生成的handler是这种类型
* HttpRequestHandler：如果handler是HttpRequestHandler就用这个
* Controller：
* Servlet:直接通过Servlet来处理

HandlerAdapter有5个具体的实现：

* HandlerFunctionAdapter:WebFlux相关，还不懂，先放一放
* HttpRequestHandlerAdapter：如果handler是HttpRequestHandler就用这个
* RequestMappingHandlerAdapter：如果handler是一个HandlerMethod就用这个
* SimpleControllerHandlerAdapter：如果handler是Controller(是类，不是@Controller注解)就用这个(这个应该已经弃用了，我看了spring的代码Annotate，Controller相关的是2008年写的。。。)
* SimpleServletHandlerAdapter：如果handler是一个javax.servlet.Servlet,那就用这个

## 实现

### RequestMappingHandlerAdapter+HandlerMethod的实现

主要逻辑在RequestMappingHandlerAdapter的invokeHandlerMethod方法中

```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
	//封装原始的request和response
  ServletWebRequest webRequest = new ServletWebRequest(request, response);
  try {
    //关于DataBinder详情见https://blog.csdn.net/yuxiuzhiai/article/details/103519070
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