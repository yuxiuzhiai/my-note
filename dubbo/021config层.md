# 概念

在dubbo的基本使用上，客户端代码可以参与的基本就是dubbo的各种配置了(SPI扩展另说)。

dubbo的config层，就对应着源码中的dubbo-config模块。dubbo-config内部又分为了：

* dubbo-config-api：具体的配置项承载model等
* dubbo-config-spring：包含了一些与spring的继承，主要就是xml或者注解方式配置dubbo时，需要提供的spring相关的一些组件



# 组件

## 类图

![](/Users/didi/workspace/study/my-note/pic/ServiceConfig.png)

可以看到，从方法 -> 接口 -> 服务提供者/消费者，是一层层的继承下来的。

# dubbo配置与spring的集成

dubbo配置集成到spring的配置，有两种方式，一种是通过xml配置，一种是通过注解配置。

## xml配置

我们知道，spring中，如果想实现自定义标签的解析，需要在spring指定的文件spring.schemas/spring.handlers中加入自己定义的xsd文件路径和对应的NamespaceHandler解析器。xsd文件，是对自定义标签的全方位的定义和约束，而handler就是解析自定义标签时用的。

在dubbo中，自定义的NamespaceHandler就是DubboNamespaceHandler。进入方法：

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {

    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }
    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
}
```

主要的逻辑都在DubboBeanDefinitionParser.parse()方法，进入

```java
private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
  RootBeanDefinition beanDefinition = new RootBeanDefinition();
  beanDefinition.setBeanClass(beanClass);
  beanDefinition.setLazyInit(false);
  String id = element.getAttribute("id");
  if ((id == null || id.length() == 0) && required) {
    String generatedBeanName = element.getAttribute("name");
    if (generatedBeanName == null || generatedBeanName.length() == 0) {
      if (ProtocolConfig.class.equals(beanClass)) {
        generatedBeanName = "dubbo";
      } else {
        generatedBeanName = element.getAttribute("interface");
      }
    }
    if (generatedBeanName == null || generatedBeanName.length() == 0) {
      generatedBeanName = beanClass.getName();
    }
    id = generatedBeanName;
    int counter = 2;
    while (parserContext.getRegistry().containsBeanDefinition(id)) {
      id = generatedBeanName + (counter++);
    }
  }
	//  
  if (id != null && id.length() > 0) {
    if (parserContext.getRegistry().containsBeanDefinition(id)) {
      throw new IllegalStateException("Duplicate spring bean id " + id);
    }
    parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
    beanDefinition.getPropertyValues().addPropertyValue("id", id);
  }
  if (ProtocolConfig.class.equals(beanClass)) {
    for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
      BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
      PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
      if (property != null) {
        Object value = property.getValue();
        if (value instanceof ProtocolConfig && id.equals(((ProtocolConfig) value).getName())) {
          definition.getPropertyValues().addPropertyValue("protocol", new RuntimeBeanReference(id));
        }
      }
    }
  } else if (ServiceBean.class.equals(beanClass)) {
    String className = element.getAttribute("class");
    if (className != null && className.length() > 0) {
      RootBeanDefinition classDefinition = new RootBeanDefinition();
      classDefinition.setBeanClass(ReflectUtils.forName(className));
      classDefinition.setLazyInit(false);
      parseProperties(element.getChildNodes(), classDefinition);
      beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
    }
  } else if (ProviderConfig.class.equals(beanClass)) {
    parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
  } else if (ConsumerConfig.class.equals(beanClass)) {
    parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);
  }
  Set<String> props = new HashSet<String>();
  ManagedMap parameters = null;
  for (Method setter : beanClass.getMethods()) {
    String name = setter.getName();
    if (name.length() > 3 && name.startsWith("set")
        && Modifier.isPublic(setter.getModifiers())
        && setter.getParameterTypes().length == 1) {
      Class<?> type = setter.getParameterTypes()[0];
      String propertyName = name.substring(3, 4).toLowerCase() + name.substring(4);
      String property = StringUtils.camelToSplitName(propertyName, "-");
      props.add(property);
      Method getter = null;
      try {
        getter = beanClass.getMethod("get" + name.substring(3), new Class<?>[0]);
      } catch (NoSuchMethodException e) {
        try {
          getter = beanClass.getMethod("is" + name.substring(3), new Class<?>[0]);
        } catch (NoSuchMethodException e2) {
        }
      }
      if (getter == null
          || !Modifier.isPublic(getter.getModifiers())
          || !type.equals(getter.getReturnType())) {
        continue;
      }
      if ("parameters".equals(property)) {
        parameters = parseParameters(element.getChildNodes(), beanDefinition);
      } else if ("methods".equals(property)) {
        parseMethods(id, element.getChildNodes(), beanDefinition, parserContext);
      } else if ("arguments".equals(property)) {
        parseArguments(id, element.getChildNodes(), beanDefinition, parserContext);
      } else {
        String value = element.getAttribute(property);
        if (value != null) {
          value = value.trim();
          if (value.length() > 0) {
            if ("registry".equals(property) && RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(value)) {
              RegistryConfig registryConfig = new RegistryConfig();
              registryConfig.setAddress(RegistryConfig.NO_AVAILABLE);
              beanDefinition.getPropertyValues().addPropertyValue(property, registryConfig);
            } else if ("registry".equals(property) && value.indexOf(',') != -1) {
              parseMultiRef("registries", value, beanDefinition, parserContext);
            } else if ("provider".equals(property) && value.indexOf(',') != -1) {
              parseMultiRef("providers", value, beanDefinition, parserContext);
            } else if ("protocol".equals(property) && value.indexOf(',') != -1) {
              parseMultiRef("protocols", value, beanDefinition, parserContext);
            } else {
              Object reference;
              if (isPrimitive(type)) {
                if ("async".equals(property) && "false".equals(value)
                    || "timeout".equals(property) && "0".equals(value)
                    || "delay".equals(property) && "0".equals(value)
                    || "version".equals(property) && "0.0.0".equals(value)
                    || "stat".equals(property) && "-1".equals(value)
                    || "reliable".equals(property) && "false".equals(value)) {
                  // backward compatibility for the default value in old version's xsd
                  value = null;
                }
                reference = value;
              } else if ("protocol".equals(property)
                         && ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(value)
                         && (!parserContext.getRegistry().containsBeanDefinition(value)
                             || !ProtocolConfig.class.getName().equals(parserContext.getRegistry().getBeanDefinition(value).getBeanClassName()))) {
                if ("dubbo:provider".equals(element.getTagName())) {
                  logger.warn("Recommended replace <dubbo:provider protocol=\"" + value + "\" ... /> to <dubbo:protocol name=\"" + value + "\" ... />");
                }
                // backward compatibility
                ProtocolConfig protocol = new ProtocolConfig();
                protocol.setName(value);
                reference = protocol;
              } else if ("onreturn".equals(property)) {
                int index = value.lastIndexOf(".");
                String returnRef = value.substring(0, index);
                String returnMethod = value.substring(index + 1);
                reference = new RuntimeBeanReference(returnRef);
                beanDefinition.getPropertyValues().addPropertyValue("onreturnMethod", returnMethod);
              } else if ("onthrow".equals(property)) {
                int index = value.lastIndexOf(".");
                String throwRef = value.substring(0, index);
                String throwMethod = value.substring(index + 1);
                reference = new RuntimeBeanReference(throwRef);
                beanDefinition.getPropertyValues().addPropertyValue("onthrowMethod", throwMethod);
              } else if ("oninvoke".equals(property)) {
                int index = value.lastIndexOf(".");
                String invokeRef = value.substring(0, index);
                String invokeRefMethod = value.substring(index + 1);
                reference = new RuntimeBeanReference(invokeRef);
                beanDefinition.getPropertyValues().addPropertyValue("oninvokeMethod", invokeRefMethod);
              } else {
                if ("ref".equals(property) && parserContext.getRegistry().containsBeanDefinition(value)) {
                  BeanDefinition refBean = parserContext.getRegistry().getBeanDefinition(value);
                  if (!refBean.isSingleton()) {
                    throw new IllegalStateException("The exported service ref " + value + " must be singleton! Please set the " + value + " bean scope to singleton, eg: <bean id=\"" + value + "\" scope=\"singleton\" ...>");
                  }
                }
                reference = new RuntimeBeanReference(value);
              }
              beanDefinition.getPropertyValues().addPropertyValue(propertyName, reference);
            }
          }
        }
      }
    }
  }
  NamedNodeMap attributes = element.getAttributes();
  int len = attributes.getLength();
  for (int i = 0; i < len; i++) {
    Node node = attributes.item(i);
    String name = node.getLocalName();
    if (!props.contains(name)) {
      if (parameters == null) {
        parameters = new ManagedMap();
      }
      String value = node.getNodeValue();
      parameters.put(name, new TypedStringValue(value, String.class));
    }
  }
  if (parameters != null) {
    beanDefinition.getPropertyValues().addPropertyValue("parameters", parameters);
  }
  return beanDefinition;
}
```

## 注解配置



# 实现

# 结语



(参考丁威、周继峰<<RocketMQ技术内幕>>。水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

