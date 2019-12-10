## 基本介绍

BeanFactory就是spring中用于存储bean的容器

## 用法

以一个已经过时了的类XmlBeanFactory来开始：

```java
package com.pk.study.spring.bean;

import com.pk.study.spring.MyBean;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.xml.XmlBeanFactory;
import org.springframework.core.io.ClassPathResource;

import static org.junit.jupiter.api.Assertions.assertEquals;

public class BeanFactoryTest {

    @Test
    public void testSimpleLoad() {
      	//加载这个xml配置文件，创建一个BeanFactory
        BeanFactory bf = new XmlBeanFactory(new ClassPathResource("beanFactory.xml"));
      	//获取Bean
        MyBean bean = (MyBean) bf.getBean("myBean");
        assertEquals("testStr", bean.getTestStr());
    }
}
```

## 实现分析

先看XmlBeanFactory的构造方法:

```java
public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
  //如果不为parenBeanFactory不为空，就给父类设置下
  super(parentBeanFactory);
  //解析xml文件
  this.reader.loadBeanDefinitions(resource);//@1
}
```

可以知道，主要解析xml的逻辑是在@1中实现的，这里的reader是XmlBeanDefinitionReader类型，那这个XmlBeanDefinitionReader是干啥的呢？答案是是读取BeanDefinition的。那BeanDefinition又是啥呢？请见：

