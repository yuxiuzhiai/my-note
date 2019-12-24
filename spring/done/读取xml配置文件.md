# 概念

在较早的spring版本中，xml是配置spring唯一的方式。在如今的spring5.x版本已经spring boot2.x版本中，xml已经不再是唯一的配置手段了，甚至已经不再是推荐的手段。

但是，作为spring元老级的功能，xml配置的方式在可预见的时间内还是不会被淘汰的。所以学习一下spring中读取xml配置文件的方法也还是不错的。

# 用法

演示一个十分基础的用法，作为讲解原理的起点。

```java
public class XmlBeanDefinitionTest {
  @Test
  public void test(){
    //创建一个实现了BeanDefinitionRegistry的BeanFactory实现
    //DefaultListableBeanFactory也是绝大多数场景下，BeanFactory的具体实现
    DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
    //XmlBeanDefinitionReader创建，从名字可以看出来 这个类是用来从xml文件中读取配置的
    XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanFactory);
    //具体解析xml文件中的配置，并注册到BeanDefinitionRegistry中去
    reader.loadBeanDefinitions(new ClassPathResource("xmlBeanDefinition.xml"));
  }
}
```

# 实现

进入reader.loadBeanDefinitions(new ClassPathResource("xmlBeanDefinition.xml"))

可以发现主要的逻辑在XmlBeanDefinitionReader.doLoadBeanDefinitions()方法中

```java
//省略异常捕获代码
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource) throws BeanDefinitionStoreException {
  //将xml文件转化为Document对象
  Document doc = doLoadDocument(inputSource, resource);//@1
  //解析配置
  int count = registerBeanDefinitions(doc, resource);//@2
  if (logger.isDebugEnabled()) {
    logger.debug("Loaded " + count + " bean definitions from " + resource);
  }
  return count;
}
```

## 1.将xml文件转换为Document对象

对应上面@1处的方法

```java
protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
  return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
                                          getValidationModeForResource(resource), isNamespaceAware());
}
```

### EntityResolver是啥

可以当做是xml文件转换为对象的一个验证模板，在spring 中有如下实现：

* BeansDtdResolver：spring中以dtd验证模式来加载xml中的<beans></beans>中的配置。会从当前路径下找spring-beans.dtd文件

* PluggableSchemaResolver：spring中以scheme模式(xsd)加载配置。这个是可插入式的，spring中用的最多的是<beans />标签。这个EntityResolver应用了SPI的手法，在使用时，会先到META-INF/spring.schemas路径下读取所有的配置项，每个配置项就是类似这种:

  ```properties
  http\://www.springframework.org/schema/beans/spring-beans.xsd=org/springframework/beans/factory/xml/spring-beans.xsd
  ```

  将一个xsd的url映射到本地的类路径下

* DelegatingEntityResolver：内部聚合了一个dtdResolver和一个schemaResolver，分别用来处理dtd模式和xsd模式的xml文件

* ResourceEntityResolver：继承自DelegatingEntityResolver，首先用DelegatingEntityResolver去读取验证文件，如果没有读取到，则在当前路径下找

### validationMode又是啥

validationMode就是上述的dtd或者xsd，是xml验证的不同模式。在spring中，使用XmlValidationModeDetector来获取验证模式，检测方法就是判断验证文件是否包含DOCTYPE，如果包含就是dtd，否则就是xsd

### DocumentLoader.loadDocument()方法

```java
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
                             ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

  DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
  if (logger.isTraceEnabled()) {
    logger.trace("Using JAXP provider [" + factory.getClass().getName() + "]");
  }
  DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
  return builder.parse(inputSource);
}
```

spring中也是通过SAX解析XML文档，项创建一个DocumentBuilderFactory，在创建出DocumentBuilder，然后去解析InputSource的文件，返回一个Document对象。具体知识都是xml的，跟spring无关，这里就不再继续介绍了

## 2.关键：将Document文档转化为BeanDefinition，并注册到BeanDefinitionRegistry中去

对应上面的@2方法

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
  //BeanDefinitionDocumentReader用于具体解析Document
  BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
  int countBefore = getRegistry().getBeanDefinitionCount();
  //解析并注册BeanDefinition
  documentReader.registerBeanDefinitions(doc, createReaderContext(resource));//@2.1
  return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

### 先看看这个createReaderContext()都干了啥

```java
public XmlReaderContext createReaderContext(Resource resource) {
   return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
         this.sourceExtractor, this, getNamespaceHandlerResolver());
}
```

#### 这个NamespaceHandlerResolver又是啥

```java
public interface NamespaceHandlerResolver {
  //将一个标签的解析映射到一个NamespaceHandler
	NamespaceHandler resolve(String namespaceUri);
}
```

作用就是根据一个标签找到一个具体的NamespaceHandler，由这个NamespaceHandler去具体解析标签。其默认实现DefaultNamespaceHandlerResolver也使用了SPI机制，会去classpath低下找META-INF/spring.handlers文件。比如大dubbo的jar中就有这个文件：

```properties
http\://dubbo.apache.org/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
```

就是把<dubbo />标签交给DubboNamespaceHandler去解析

### 2.1.DocumentReader.registerBeanDefinitions()方法

进入@2.1处的方法，然后兜兜转转，找到具体的逻辑处理方法：DocumentReader.doRegisterBeanDefinition()方法

然后再找到更具体的处理方法😂，DocumentReader.parseBeanDefinitions()

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
            //@2.1.1解析默认标签
						parseDefaultElement(ele, delegate);
					}
					else {
            //@2.1.2解析自定义标签
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
```

转来转去，重要找到了，真的去处理xml里面各个标签的方法了。

#### 解析默认标签

@4的方法用于解析默认标签。

主要的逻辑在DefaultBeanDefinitionDocumentReader.parseDefaultElement()方法中

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
  //解析import标签
  if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
    importBeanDefinitionResource(ele);
  }
  //解析alias标签
  else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
    processAliasRegistration(ele);
  }
  //解析bean标签
  else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
    processBeanDefinition(ele, delegate);
  }
  //解析嵌套的beans标签
  else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
    // recurse
    doRegisterBeanDefinitions(ele);
  }
}
```

##### 解析import标签

一般使用如：<import resource="xx.xml" />

用于导入别的配置文件。其解析非常简单：

* 找到对应的文件
* 调用前面的reader的loadBeanDefinitions方法

##### 解析alias标签

类似这样

```xml
<bean id="userInfo" class="com.pk.study.spring.UserInfo"/>
<alias name="userInfo" alias="hahaha,aaa"/>
```

解析也很简单，就是调用AliasRegistry接口的registerAlias方法注册一下

##### 解析bean标签

这个是最复杂的标签，有茫茫多的属性配置。所以我单独用一篇文章来分析😂

请见：https://blog.csdn.net/yuxiuzhiai/article/details/103540807

#### 2.1.2.解析自定义标签

如果还不清楚如果在spring中使用自定义的标签，可以看下这篇：

# 结语

在spring中，加载xml文件，着实是非常复杂了。。。

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。如有错误，请指正~)