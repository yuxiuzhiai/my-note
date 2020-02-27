## 概念

locale即语言区域，或者说就是国际化相关。在spring mvc中，还有个LocaleContext的概念，根据时区来确定locale

## 实现

在spring mvc中，使用了LocaleResolver接口来处理Locale。

```java
//最淳朴的LocalResolver接口就仅仅有locale的get set
public interface LocaleResolver {
	Locale resolveLocale(HttpServletRequest request);
	void setLocale(HttpServletRequest request, @Nullable HttpServletResponse response, @Nullable Locale locale);
}
//LocalContext是决定locale的一个策略接口
public interface LocaleContextResolver extends LocaleResolver {
	LocaleContext resolveLocaleContext(HttpServletRequest request);
	void setLocaleContext(HttpServletRequest request, @Nullable HttpServletResponse response,
			@Nullable LocaleContext localeContext);

}
```

依次看看LocalResolver的具体实现：

* AcceptHeaderLocaleResolver：根据请求头部的Accept-Language字段决定locale，如果请求没有这个首部字段，就用服务器的默认locale
* CookieLocaleResolver：通过在cookie中设置好locale，然后后续的locale就用cookie中的
* FixedLocaleResolver：永远返回一个固定的locale，默认是服务端的locale
* SessionLocaleResolver：通过在session中添加一个locale字段实现

## 结语

在spring mvc中，locale的相关处理的相关类有：

* LocaleResolver：8个
* LocalContext：4个
* ocaleContextHolder：将一个LocalContext关联到当前线程的holder 工具类


(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。如有错误，请指正~)