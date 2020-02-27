# 介绍

aop作为oop的补充，一般用于实现一些公共逻辑，比如日志、事务等。aop可以使这些公共逻辑与具体的业务逻辑解耦，从而使应用开发者可以更加专注于业务代码的开发，代码也可以更加的简洁优雅。大体上aop有两种类型：

* 动态aop:通过动态代理技术，比如cglib或者jdk自带的动态代理，在运行时通过动态生成的字节码替换原有的实现
* 静态织入：通过java.lang.instrument实现一个Java agent，在虚拟机启动时，就通过改变目标对象字节码的方式来完成对目标对象的增强

# 术语

## 连接点Joint point

表示在程序中明确定义的点，典型的包括方法调用，对类成员的访问以及异常处理程序块的执行等等，它自身还可以嵌套其它连接点jointpoint。在spring aop中，可以就理解成是方法执行

## 切点Pointcut

表示一组连接点jointpoint，这些连接点或是通过逻辑关系组合起来，或是通过通配、正则表达式等方式集中起来，它定义了相应的操作处理通知Advice将要发生的地方

spring中对应的接口：

```java
public interface Pointcut {
	//类过滤器，用于判定一个类是否符合Pointcut的定义
  ClassFilter getClassFilter();
	//方法匹配器，用于判定一个方法是否符合Pointcut定义
  MethodMatcher getMethodMatcher();

  Pointcut TRUE = TruePointcut.INSTANCE;
}
```

## 通知Advice

Advice 定义了在切入点pointcut 里面定义的程序点具体要做的操作和处理，它通过 before、after 和 around 来区别是在每个切入点之前、之后还是代替执行的代码

在spring中对应Advice接口(空接口)。它的几个子接口(都是标记接口)：

* BeforeAdvice：方法执行前执行
* AfterAdvice：方法执行后
* AroundAdvice：环绕
* ThrowsAdvice：方法抛异常时执行

## spring中定义的Advisor

持有advice和advice的使用过滤器(比如Pointcut)的基础接口



## 实现

## 结语

在spring中，spring.factories文件是很重要的。所以虽然很简单，但肯定要知道

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。如有错误，请指正~)