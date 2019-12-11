## 概念

我们知道在spring mvc中，当来了个请求的时候，首先要根据HandlerMapping去找到handler，然后就是要去执行这个handler，但是handler的种类不同，自然也需要不同的执行方式。这里的HandlerAdapter就可以当成是handler的执行方式

## 用法

handler个几种类型

* HandlerMethod：基于@Controller和@RequestMapping注解生成的handler是这种类型
* HttpRequestHandler：如果handler是HttpRequestHandler就用这个
* RequestMappingHandlerAdapter：如果handler是一个HandlerMethod就用这个
* SimpleControllerHandlerAdapter：如果handler是Controller(是类，不是@Controller注解)就用这个(这个应该已经弃用了，我看了spring的代码Annotate，Controller相关的是2008年写的。。。)
* SimpleServletHandlerAdapter：如果handler是一个javax.servlet.Servlet,那就用这个

## 实现

## 结语

在spring中，spring.factories文件是很重要的。所以虽然很简单，但肯定要知道

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。
如有错误，请指正~)