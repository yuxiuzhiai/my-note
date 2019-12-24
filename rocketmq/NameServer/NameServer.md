# 概念

在rocketmq中，NameServer用于Broker的注册与发现。相当于kafka中，zookeeper的角色。不过，相较于zookeeper，NameServer是专用于根据消息队列这一场景下的特定实现。所以，实现上更加的优雅和简洁。

# 组件



# 用法



# 实现

主要分析两个过程。NameServer的启动；NameServer对路由信息的管理

## 启动



# 结语



(水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

