# 概念

broker作为rocketmq的核心枢纽，无疑是最重要的部分。在rocketmq中，无论是namesrv还是broker，其使用方式大概相同，先读取配置，主要功能由对应的xxController统一封装，xxController是其几乎所有内部功能组件的容器。



# 组件

## ConsumerOffsetManager

## ConsumerManager

## ConsumerFilterManager

## ProducerManager

## ClientHousekeepingService

是一个ChannelEventListener，主要就是会监听channel的事件，如果channel关闭或者异常，broker会把对应的consumer、producer、filterServer也关闭；另外会启动一个定时任务，定时扫描无效的consumer、producer和filterServer。

## BrokerOuterAPI

在broker之间，或者是broker和namesrv之间发送请求，比如心跳之类的

# 作用

在rocketmq中，或者在别的其他消息中间件中，broker的作用一般可以大体分为：

* 作为服务端，接收producer发送的消息
* 持久化消息
* 作为消费端，响应consumer消费消息的请求
* 作为中间件服务端，本身需要一定的ha机制保障高可用
* 维持与客户端的心跳

# 实现

本文会逐个分析下列过程：

* broker的启动
* broker接收消息的处理

## broker的启动

broker的启动和namesrv的启动很类似，先创建一个controller，然后启动这个controller。BrokerController基本就是一个所有broker组件的容器。每个组件实现一部分功能，其实对具体功能实现的分析，就是对某个组件的实现分析。

### 1.创建controller

与namesrv不同的时，controller有四个大项的配置：

* BrokerConfig：作为broker本身的一些配置
* NettyServerConfig：作为netty服务端的配置
* NettyClientConfig：对netty客户端的配置
* MessageStoreConfig：消息持久化的相关配置

新建一个BrokerController后，会调用她的initilize()方法：

```java
public boolean initialize() throws CloneNotSupportedException {
  boolean result = this.topicConfigManager.load();
  result = result && this.consumerOffsetManager.load();
  result = result && this.subscriptionGroupManager.load();
  result = result && this.consumerFilterManager.load();
  if (result) {
    //配置DefaultMessageStore，这个类是消息持久化的核心，后面分单独介绍
    this.messageStore = ...
  }
  result = result && this.messageStore.load();
  if (result) {
    //启动一个监听客户端连接的netty服务端
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.clientHousekeepingService);
    //启动一个vip通道
    NettyServerConfig fastConfig = (NettyServerConfig) this.nettyServerConfig.clone();
    fastConfig.setListenPort(nettyServerConfig.getListenPort() - 2);
    this.fastRemotingServer = new NettyRemotingServer(fastConfig, this.clientHousekeepingService);
    //后面的这些个线程池配置都会配置到netty上用于处理特定的请求
    //
    this.sendMessageExecutor = ..
		//
    this.pullMessageExecutor = ..
		//配置响应消息的线程池
    this.replyMessageExecutor = ..
		//配置消息查询的线程池
    this.queryMessageExecutor = ..
		//admin相关的线程池
    this.adminBrokerExecutor = ..
		//
    this.clientManageExecutor = ..
		//心跳管理的线程池
    this.heartbeatExecutor = ..
		//事务线程池
    this.endTransactionExecutor = ..
		//消费者管理线程池
    this.consumerManageExecutor = ..
    //注册请求处理器，各种NettyRequestProcessor，用于处理不同的客户端请求
    this.registerProcessor();

    final long initialDelay = UtilAll.computeNextMorningTimeMillis() - System.currentTimeMillis();
    final long period = 1000 * 60 * 60 * 24;
    //定时任务，记录broker状态
    this.scheduledExecutorService.scheduleAtFixedRate(..);
		//定时任务，每10s持久化消费者消费进度
    this.scheduledExecutorService.scheduleAtFixedRate(..);
		//定时任务，每10s持久化消费者消费过滤器
    this.scheduledExecutorService.scheduleAtFixedRate(..);
		//定时任务，每3min持久化消费者消费过滤器
    this.scheduledExecutorService.scheduleAtFixedRate(..);
		//定时任务，每1s打印系统负载
    this.scheduledExecutorService.scheduleAtFixedRate(..);

    if (this.brokerConfig.getNamesrvAddr() != null) {
      this.brokerOuterAPI.updateNameServerAddressList(this.brokerConfig.getNamesrvAddr());
      log.info("Set user specified name server address: {}", this.brokerConfig.getNamesrvAddr());
    } else if (this.brokerConfig.isFetchNamesrvAddrByAddressServer()) {
      //定时任务，每2min获取一次namesrv
      this.scheduledExecutorService.scheduleAtFixedRate(..);
    }

    if (!messageStoreConfig.isEnableDLegerCommitLog()) {
      if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
        if (this.messageStoreConfig.getHaMasterAddress() != null && this.messageStoreConfig.getHaMasterAddress().length() >= 6) {
          this.messageStore.updateHaMasterAddress(this.messageStoreConfig.getHaMasterAddress());
          this.updateMasterHAServerAddrPeriodically = false;
        } else {
          this.updateMasterHAServerAddrPeriodically = true;
        }
      } else {
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
          @Override
          public void run() {
            try {
              BrokerController.this.printMasterAndSlaveDiff();
            } catch (Throwable e) {
              log.error("schedule printMasterAndSlaveDiff error.", e);
            }
          }
        }, 1000 * 10, 1000 * 60, TimeUnit.MILLISECONDS);
      }
    }

    if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
      //SslContext相关
      fileWatchService = ..
    }
    initialTransaction();
    initialAcl();
    initialRpcHooks();
  }
  return result;
}
```

### 2.启动BrokerController

代码如下：

```java
public void start() throws Exception {

  this.messageStore.start();
	//netty服务端，监听客户端连接，处理客户端请求。fastRemotingServer跟remotingServer没啥区别，相当于一个vip通道，消息发送的时候可以往这个vip通道的端口上发送消息
  this.remotingServer.start();
  this.fastRemotingServer.start();
	//监听配置文件的改动
  this.fileWatchService.start();
	//
  this.brokerOuterAPI.start();
	//
  this.pullRequestHoldService.start();
	//
  this.clientHousekeepingService.start();
	//
  this.filterServerManager.start();

  if (!messageStoreConfig.isEnableDLegerCommitLog()) {
    startProcessorByHa(messageStoreConfig.getBrokerRole());
    handleSlaveSynchronize(messageStoreConfig.getBrokerRole());
    this.registerBrokerAll(true, false, true);
  }

  this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
      try {
        BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
      } catch (Throwable e) {
        log.error("registerBrokerAll Exception", e);
      }
    }
  }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);
  this.brokerStatsManager.start();
  this.brokerFastFailure.start();
}
```

## broker接收消息的处理



# 结语



(参考丁威、周继峰<<RocketMQ技术内幕>>。水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

