## 概念

spring解析xml配置文件是一件十分复杂的事情，而解析bean标签又是这件复杂的事情里面更加复杂的事情。

## 实现

具体方法是DefaultBeanDefinitionDocumentReader.processBeanDefinition()方法

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
  ////@1.关键方法，将Element解析成BeanDefinitionHolder
  BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
  if (bdHolder != null) {
    //@2.处理自定义的子元素标签
    bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
    try {
      //@3.注册BeanDefinition到BeanDefinitionRegistry
      BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
    }
    catch (BeanDefinitionStoreException ex) {
      getReaderContext().error("Failed to register bean definition with name '" +
                               bdHolder.getBeanName() + "'", ele, ex);
    }
    //@4.触发一个事件，实际上啥都没做
    getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
  }
}
```



### 1.将Element转化为BeanDefinitionHolder

一层一层下来，终于看到具体的解析标签的逻辑了

```java
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
  //解析id、name属性
  String id = ele.getAttribute(ID_ATTRIBUTE);
  String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
  List<String> aliases = new ArrayList<>();
  if (StringUtils.hasLength(nameAttr)) {
    //name会按逗号分隔，用于后面注册到AliasRegistry中去
    String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
    aliases.addAll(Arrays.asList(nameArr));
  }

  String beanName = id;
  if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
    beanName = aliases.remove(0);
    if (logger.isTraceEnabled()) {
      logger.trace("No XML 'id' specified - using '" + beanName +
                   "' as bean name and " + aliases + " as aliases");
    }
  }

  if (containingBean == null) {
    checkNameUniqueness(beanName, aliases, ele);
  }
	//具体逻辑
  AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);//@1.1
  if (beanDefinition != null) {
    if (!StringUtils.hasText(beanName)) {
      try {
        if (containingBean != null) {
          beanName = BeanDefinitionReaderUtils.generateBeanName(
            beanDefinition, this.readerContext.getRegistry(), true);
        }
        else {
          beanName = this.readerContext.generateBeanName(beanDefinition);
          // Register an alias for the plain bean class name, if still possible,
          // if the generator returned the class name plus a suffix.
          // This is expected for Spring 1.2/2.0 backwards compatibility.
          String beanClassName = beanDefinition.getBeanClassName();
          if (beanClassName != null &&
              beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
              !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
            aliases.add(beanClassName);
          }
        }
        if (logger.isTraceEnabled()) {
          logger.trace("Neither XML 'id' nor 'name' specified - " +
                       "using generated bean name [" + beanName + "]");
        }
      }
      catch (Exception ex) {
        error(ex.getMessage(), ele);
        return null;
      }
    }
    String[] aliasesArray = StringUtils.toStringArray(aliases);
    //包装成BeanDefinitionHolder
    return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);//@1.2
  }

  return null;
}
```

#### 1.1.创建BeanDefinition

继续看看@1.1处的方法

```java
public AbstractBeanDefinition parseBeanDefinitionElement(Element ele, String beanName, @Nullable BeanDefinition containingBean) {

  this.parseState.push(new BeanEntry(beanName));

  String className = null;
  if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
    className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
  }
  String parent = null;
  if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
    parent = ele.getAttribute(PARENT_ATTRIBUTE);
  }
	//省略了try catch
  
  //创建一个GenericBeanDefinition
  AbstractBeanDefinition bd = createBeanDefinition(className, parent);
  //解析一些常见的简单属性，操作就是在beanDefinition中设置对应的值，没有复杂操作
  parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
  //提取子标签description
  bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
  //解析元数据，即<meta />标签
  parseMetaElements(ele, bd);
  //解析lookup-method属性
  parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
  //解析replaced-method属性
  parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
  //解析构造函数参数
  parseConstructorArgElements(ele, bd);
  //解析property子元素
  parsePropertyElements(ele, bd);
  //解析qualified子元素
  parseQualifierElements(ele, bd);

  bd.setResource(this.readerContext.getResource());
  bd.setSource(extractSource(ele));

  return bd;
}
```

#### 创建一个BeanDefinitionHolder

将BeanDefinition结合beanName以及alias，生成一个BeanDefinitionHolder返回

### 2.默认标签中的自定义子标签处理

@2处的代码，用于处理bean标签下可能存在的自定义子标签

```java
public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {

  BeanDefinitionHolder finalDefinition = originalDef;
  // Decorate based on custom attributes first.
  NamedNodeMap attributes = ele.getAttributes();
  for (int i = 0; i < attributes.getLength(); i++) {
    Node node = attributes.item(i);
    finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
  }

  // Decorate based on custom nested elements.
  NodeList children = ele.getChildNodes();
  for (int i = 0; i < children.getLength(); i++) {
    Node node = children.item(i);
    if (node.getNodeType() == Node.ELEMENT_NODE) {
      finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
    }
  }
  return finalDefinition;
}
```

### 3.注册BeanDefinition

将BeanDefinition注册到BeanDefinitionRegistry中；

将alias注册到AliasRegistry中

### 4.触发一个事件

啥都没干

## 结语

本文阐述了spring通过xml作为配置文件时，具体解析一个bean标签的原理与过程。过程复杂，方法嵌套层级深。加油~

(水平有限，最近在看spring源码，分享学习过程，希望对各位有点微小的帮助。如有错误，请指正~)