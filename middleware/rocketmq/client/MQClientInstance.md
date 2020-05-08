# 概念

# 作用

MQClientInstance的主要作用，就是提供一些客户端的公共的基础功能，比如定时向namesrv更新路由信息，定时向broker发送心跳信息。这种功能无论是消费者还是生产者都需要，所以需要一个公共的组件来实现，就是MQClientInstance。

# 组件

## MQClientAPIImpl



# 实现

（列出的代码删减了部分不太重要的部分）

## 启动

先看看启动部分的代码：MQClientInstance.start()

```java
public void start() throws MQClientException {
  //如果没有配置namesrv地址，则尝试请求一个配置项指定的地址获取
  if (null == this.clientConfig.getNamesrvAddr()) {
    this.mQClientAPIImpl.fetchNameServerAddr();
  }
  // Start request-response channel
  this.mQClientAPIImpl.start();
  //启动各种定时任务
  this.startScheduledTask();
  //启动PullMessageService，是消费者需要的服务
  this.pullMessageService.start();
  //启动RebalanceService，是生产者需要的服务
  this.rebalanceService.start();
  //启动生产者
  this.defaultMQProducer.getDefaultMQProducerImpl().start(false); 
}
```

重点看下启动的各个定时任务：

```java
private void startScheduledTask() {
  //如果启动时没有指定namesrv地址，则启动定时任务，每2min获取一次
  if (null == this.clientConfig.getNamesrvAddr()) {
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
      public void run() {
        MQClientInstance.this.mQClientAPIImpl.fetchNameServerAddr();
      }
    }, 1000 * 10, 1000 * 60 * 2, TimeUnit.MILLISECONDS);
  }
  //每过一定时间，默认30s，从namesrv获取一次路由信息
  this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    public void run() {
      MQClientInstance.this.updateTopicRouteInfoFromNameServer();
    }
  }, 10, this.clientConfig.getPollNameServerInterval(), TimeUnit.MILLISECONDS);
	//每过一定时间，默认30s，清理一次下线的broker，并向活着的broker发送心跳
  this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    public void run() {
      MQClientInstance.this.cleanOfflineBroker();
      MQClientInstance.this.sendHeartbeatToAllBrokerWithLock();
    }
  }, 1000, this.clientConfig.getHeartbeatBrokerInterval(), TimeUnit.MILLISECONDS);
	//每过一定时间，默认5s，保存一次消费者进度
  this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    public void run() {
        MQClientInstance.this.persistAllConsumerOffset();
    }
  }, 1000 * 10, this.clientConfig.getPersistConsumerOffsetInterval(), TimeUnit.MILLISECONDS);
	//每过1min钟，调整一次线程池
  this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        MQClientInstance.this.adjustThreadPool();
    }
  }, 1, 1, TimeUnit.MINUTES);
}
```

## 获取路由信息





# 结语



(参考丁威、周继峰<<RocketMQ技术内幕>>。水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

