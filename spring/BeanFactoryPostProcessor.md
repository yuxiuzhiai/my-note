# 介绍

在spring 中，BeanPostProcessor一般用于在已经解析了各种bean，将bean转换为BeanDefinition之后，将bean实例化之前，来做一些自定义的处理。

BeanPostProcessor还有一个子接口：BeanDefinitionRegistryPostProcessor。

接口定义：

BeanPostProcessor

```java
public interface BeanFactoryPostProcessor {
	//在已经解析了bean的定义，装换为BeanDefinition之后，具体实例化bean之前，做一些自定义的事情
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}

public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
	//添加BeanDefinition，还可以用来添加BeanPostProcessor
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}
```



# spring中的具体实现

* ConfigurationClassPostProcessor：非常重要的实现

# 实现



# 结语

在spring中，spring.factories文件是很重要的。所以虽然很简单，但肯定要知道

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。如有错误，请指正~)