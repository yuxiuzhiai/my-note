## 概念

在spring中，MessageSource接口用于抽象对信息的定义，相当于是信息的提供者。比如，在web系统中，我们经常定义ResourceBundle来处理国际化相关的文案下发。在spring中，把从消息标识转化为具体的字符串的这一过程，抽象成了MessageSource接口，使可以更方便的实现消息的参数化和国际化。

## 类个层级结构

先看看MessageSource的定义：

```java
public interface MessageSource {
	//根据code，参数，默认值，locale获取消息
  String getMessage(String code, @Nullable Object[] args, @Nullable String defaultMessage, Locale locale);
	//基本跟上面方法一直，只是没有默认消息
  String getMessage(String code, @Nullable Object[] args, Locale locale) throws NoSuchMessageException;
	//
  String getMessage(MessageSourceResolvable resolvable, Locale locale) throws NoSuchMessageException;
}
```

我们再看看这个MessageSourceResolvable又是啥

```java
public interface MessageSourceResolvable {
	//code有多个，用于在第一个找不到时候，再去找第二个第三个..
  String[] getCodes();
  default Object[] getArguments() {
    return null;
  }
  default String getDefaultMessage() {
    return null;
  }
}
```

可以看到这个MessageSourceResolvable其实就是对获取message时，那几个参数的封装，不过是code可以有多个，相当于一种fallback机制。

整个类的层级结构：

<img src="/Users/didi/workspace/study/my-note/pic/MessageSource.png" style="zoom:80%;" />

## 实现

* ResourceBundleMessageSource：以一组properties文件组成的ResourceBundle来作为message的消息源
* ReloadableResourceBundleMessageSource：可以重载的message消息源。源可以是properties文件或者是xml文件
* StaticMessageSource:提供编程式添加message的方法，不依赖文件提供消息源。其实没啥用
* DelegatingMessageSource：自己不实现查找message的功能，而是通过在内部聚合一个别的MessageSource来实现功能

## 结语

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。如有错误，请指正~)