## 概念

在spring中,主题是用以解析一组特定的消息、带么、文件路径等，包括css和图片文件。其实可以类比于我们在手机上用的主题。代表着一套特定的字体啊，图标样式啊，诸如此类。

## 用法



## 实现

在spring mvc的DispatcherServlet中，会在初始化的时候创建一个ThemeResolver

```java
public interface ThemeResolver {
	//根据request获取到theme的name
  String resolveThemeName(HttpServletRequest request);
	//设置themeName
  void setThemeName(HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable String themeName);
}
```

ThemeResolver的具体实现类：

* CookieThemeResolver：在cookie中设置一个theme相关的属性
* FixedThemeResolver：返回一个固定的值，默认就是theme字符串
* SessionThemeResolver：在session中存一个theme的属性

ThemeResolver的作用在于根据http 请求获取一个theme的name

而根据name去获取到具体的Theme实现的功能，则是由ThemeSource接口提供：

```java
public interface ThemeSource {
	Theme getTheme(String themeName);
}
```

所有的web相关的ApplicationContext实现都同时也实现了ThemeSource接口，但是只是声明实现了这个接口，具体的功能实现，是由：ResourceBundleThemeSource提供的

## 结语

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。如有错误，请指正~)

