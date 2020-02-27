# 概念

在dubbo中，注册中心用于服务的注册发现机制。涉及到的流程有：

* provider启动时，会向注册中心注册自己的元数据，并且会订阅configurators信息
* consumer启动时，也会向注册中心注册自己的元数据，并且订阅providers，routers，configurators信息
* provider和consumer下线时(非kill -9)会向注册中心取消注册自己

# 类图

![](/Users/didi/workspace/study/my-note/pic/ZookeeperRegistry.png)

