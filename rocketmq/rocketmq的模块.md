# 介绍

rocketmq是大阿里开源的一个高性能消息中间件。

## 为啥我想看rocketmq的源码

因为吧，当前的消息中间件，声名远播、广为传颂的就两个：kafka和rocketmq。kafka的源码，比spring还多，光是能从源码编译运行起来就已经很费劲了，所以，我选择先看rocketmq的源码。

不得不说，相较于kafka，rocketmq的源码简直是一股清流，简洁高效，去除测试和示例代码，总共的源码类不超过700个，而且，直接就能运行，非常人性化。看了这么多的源码，只有rocketmq和redis的源码，让我觉得很舒服。

# 核心概念

## Namesrv

kafka是借助了zookeeper来管理集群元数据的，而rocketmq没有。在rocketmq中，namesrv充当了zookeeper的角色，用于管理集群和元数据。由于NameSrv是rocketmq为消息队列这一专门的场景下定制的实现，所以NameSrv的实现，相较于zookeeper也是十分的简洁高效。

namesrv与zookeeper有一个很大的不同，就是namesrv集群之间并不会相互通信，也就是说namesrv之间的元数据并不是完全一致的

## 生产者producer

## 消费者consumer

## broker

## 消息队列





# 实现

# 结语

(参考丁威、周继峰<<RocketMQ技术内幕>>。水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

