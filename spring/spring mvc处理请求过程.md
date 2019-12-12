## 概念

本文会介绍一个请求到达spring mvc的整个处理过程。

我们知道在spring mvc中，一般是会有DispatcherServlet统一处理所有的请求。接下来一起看看整个的流程

## 用法



## 实现

先看下核心类DispatcherServlet的类图

<img src="/Users/didi/workspace/study/my-note/pic/DispatcherServlet.png" style="zoom:50%;" />

所以DispatcherServlet首先是一个普通的Servlet实现，其次，它实现了ApplicationContextAware，所以可以在运行时获取到ApplicationContext，结合到spring的上下文中。所以DIspatcherServlet是Servlet跟spring相互结合的一个纽带。

### 初始化

当第一个请求到来时，Servlet需要首先执行init初始化。

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

### 请求处理

请求处理的核心逻辑在DispatcherServlet的dispatch()方法

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
      //看看请求中是不是包含了一个文件
      processedRequest = checkMultipart(request);
      multipartRequestParsed = (processedRequest != request);

      //找到处理的handler
      mappedHandler = getHandler(processedRequest);
      if (mappedHandler == null) {
        noHandlerFound(processedRequest, response);
        return;
      }

      // Determine handler adapter for the current request.
      HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

      // Process last-modified header, if supported by the handler.
      String method = request.getMethod();
      boolean isGet = "GET".equals(method);
      if (isGet || "HEAD".equals(method)) {
        long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
        if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
          return;
        }
      }

      if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        return;
      }

      // Actually invoke the handler.
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



## 结语

在spring中，spring.factories文件是很重要的。所以虽然很简单，但肯定要知道

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。
如有错误，请指正~)