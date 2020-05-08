# 概念

在rocketmq中，NameServer用于Broker的注册与发现。相当于kafka中，zookeeper的角色。不过，相较于zookeeper，NameServer是专用于根据消息队列这一场景下的特定实现。所以，实现上更加的优雅和简洁。

# 作用

* 每台broker启动时，无论是master还是slave，都需要主动向namesrv发送请求，注册自己
* producer在生产消息前，得知道broker的地址，而broker可能随时都会扩缩容，所以必然不能直接把地址写死，这时，producer也是通过给namesrv发请求，来获取自己要发送的topic的broker地址的
* consumer消费topic时，也是一样，需要先向namesrv发送请求，获取路由信息
* broker下线时，如果是正常关机也会请求namesrv，发现一个取消注册请求
* namesrv每隔10s会扫描一次向自己注册的broker，看看是否还活着

一言以蔽之，namesrv就是topic的服务注册发现机制存在的，向客户端（包括生产者和消费者）提供broker的信息。

再更具体地看一下，namesrv处理的请求类型，以及自己的定时任务

## 处理的命令

在rocketmq中，namesrv和broker会处理的各种命令，都会在各种NettyRequestProcessor中定义好，然后通过switch语句将对应的请求分发到对应的方法中去。在namesrv中，由DefaultRequestProcessor完成命令的分配。

### 挨个看一遍命令

* PUT_KV_CONFIG/GET_KV_CONFIG/DELETE_KV_CONFIG
* QUERY_DATA_VERSION:查询版本号
* REGISTER_BROKER：注册broker路由信息。每个broker在启动的时候会发送这个命令，向namesrv注册自己
* UNREGISTER_BROKER：如果broker正常关机，会发送这个命令，向namesrv取消自己的注册消息
* GET_ROUTEINTO_BY_TOPIC：根据topic获取路由信息。一般由消费者发送
* GET_BROKER_CLUSTER_INFO
* WIPE_WRITE_PERM_OF_BROKER
* GET_ALL_TOPIC_LIST_FROM_NAMESERVER
* DELETE_TOPIC_IN_NAMESRV
* GET_KVLIST_BY_NAMESPACE
* GET_TOPICS_BY_CLUSTER

* GET_SYSTEM_TOPIC_LIST_FROM_NS
* GET_UNIT_TOPIC_LIST
* GET_HAS_UNIT_SUB_TOPIC_LIST
* GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST
* UPDATE_NAMESRV_CONFIG
* GET_NAMESRV_CONFIG

这里面多数都是为了后台管理需要，跟消息生产消费强相关的只有：

* REGISTER_BROKER
* UNREGISTER_BROKER
* GET_ROUTEINTO_BY_TOPIC

## 定时任务

NamesrvController中定义了两个定时任务的处理：

* 每隔10s扫描一遍broker，看看是否还活着
* 每隔10min打印一下namesrv的kv配置

# 组件

从实现的角度，分析namesrv内部的各个组件

## NamesrvController

namesrv的主要行为组件

## NamesrvConfig

自己的一些配置参数

## NettyServerConfig

用于netty服务器的配置

## KVConfigManager

## RouteInfoManager

我们知道，namesrv的主要作用就是管理路由信息，作为一种以topic为主体的服务注册发现机制。那啥是路由信息呢?

我们来看看RouteInfoManager的几个重要属性：

* topicQueueTable:HashMap<String,List<QueueData>>：存储topic -> QueueData的映射。在rocketmq中，一个topic会有多个MessageQueue，QueueData就存储了MessageQueue的元数据
* brokerAddrTable:HashMap<String,BrokerData>:存储了brokerName -> broker元数据的映射。在rocketmq中，相同的brokerName组成一组master-slave结构
* clusterAddrTable:HashMap<String,Set<String>>:存储了集群 -> brokerName的映射。在rocketmq中，多个brokerName组成一个集群
* brokerLiveTable:HashMap<String,BrokerLiveInfo>:存储每个broker的心跳信息
* filterServerTable:HashMap<String,List<String>>:存储broker -> filter server的映射（filter server的概念好像已经从rocketmq中移除了）

虽然这个类名叫做RouteInfoManager，但是并没有一个类叫做RouteInfo，所以，所谓路由信息，可以理解为，就是由这几个Map包含的信息组成

# 实现

主要分析以下过程：

* namesrv的启动
* namesrv注册路由信息
* namesrv删除路由信息
* namesrv响应路由查询

(贴出的代码都做了一定的删减，去掉了日志、try catch等)

## namesrv的启动

namesrv的启动类是NamesrvStartup，启动方法就是main()方法，main()方法只有一句，就是调用main0()。进入main0()方法：

```java
public static NamesrvController main0(String[] args) {
  //1.创建NamesrvController
  NamesrvController controller = createNamesrvController(args);
  //2.启动NamesrvController
  start(controller);
  return controller;
}
```

很简单。先创建一个NamesrvController，然后启动这个Controller。逐一看看

### 1.创建NamesrvController

代码如下:

```java
public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
  System.setProperty(RemotingCommand.REMOTING_VERSION_KEY, Integer.toString(MQVersion.CURRENT_VERSION));
	//1.1.解析命令行参数
  Options options = ServerUtil.buildCommandlineOptions(new Options());
  commandLine = ServerUtil.parseCmdLine("mqnamesrv", args, buildCommandlineOptions(options), new PosixParser());
	//1.2.根据命令行参数，配置两个配置类NamesrvConfig、NettyServerConfig
  final NamesrvConfig namesrvConfig = new NamesrvConfig();
  final NettyServerConfig nettyServerConfig = new NettyServerConfig();
  nettyServerConfig.setListenPort(9876);
  //如果有-c就读取配置文件
  if (commandLine.hasOption('c')) {
    String file = commandLine.getOptionValue('c');
    if (file != null) {
      InputStream in = new BufferedInputStream(new FileInputStream(file));
      properties = new Properties();
      properties.load(in);
      MixAll.properties2Object(properties, namesrvConfig);
      MixAll.properties2Object(properties, nettyServerConfig);
      namesrvConfig.setConfigStorePath(file);
      in.close();
    }
  }
  //如果有-p那就直接打印一下配置，然后就退出
  if (commandLine.hasOption('p')) {
    InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
    MixAll.printObjectProperties(console, namesrvConfig);
    MixAll.printObjectProperties(console, nettyServerConfig);
    System.exit(0);
  }
  MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);
	//验证必须参数
  if (null == namesrvConfig.getRocketmqHome()) {
    System.exit(-2);
  }
	//1.3.配置日志
  LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
  JoranConfigurator configurator = new JoranConfigurator();
  configurator.setContext(lc);
  lc.reset();
  configurator.doConfigure(namesrvConfig.getRocketmqHome() + "/conf/logback_namesrv.xml");
  log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);
	//1.4.NamesrvController的创建
  final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);
  // remember all configs to prevent discard
  controller.getConfiguration().registerConfig(properties);

  return controller;
}
```

#### 1.1.解析命令行参数

借助apache的common-cli包，解析命令行参数。没啥好说的。

#### 1.2.根据命令行参数，配置两个配置类NamesrvConfig、NettyServerConfig

如果命令行有：

* -c +文件路径：则读取该配置文件，然后根据属性merge到NamesrvConfig、NettyServerConfig两个配置类中去
* -p：则仅仅打印一下配置，然后退出

然后验证，必须要有rocketmqHome配置项，可以来源于命令行或者是环境变量

#### 1.3.配置日志

看起来rocketmq是写死了用logback作为日志。会从rocketmqHome对应的目录底下，加载/conf/logback_namesrv.xml文件作为logback的配置文件，然后初始化内部的日志系统

#### 1.4.NamesrvController的创建

构造函数如下：

```java
public NamesrvController(NamesrvConfig namesrvConfig, NettyServerConfig nettyServerConfig) {
  this.namesrvConfig = namesrvConfig;
  this.nettyServerConfig = nettyServerConfig;
  this.kvConfigManager = new KVConfigManager(this);
  this.routeInfoManager = new RouteInfoManager();
  this.brokerHousekeepingService = new BrokerHousekeepingService(this);
  this.configuration = new Configuration(log,this.namesrvConfig, this.nettyServerConfig);
  this.configuration.setStorePathFromConfig(this.namesrvConfig, "configStorePath");
}
```

### 2.启动controller

代码如下：

```java
public static NamesrvController start(final NamesrvController controller) throws Exception {
  //2.1.初始化NamesrvController
  boolean initResult = controller.initialize();
  //2.2.注册jvm的hook函数
  Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
    @Override
    public Void call() throws Exception {
      controller.shutdown();
      return null;
    }
  }));
  //2.3.启动controller
  controller.start();
  return controller;
}
```

#### 2.1.初始化controller

```java
public boolean initialize() {
	//加载kv配置
  this.kvConfigManager.load();
  //1.创建一个Netty服务端对象，注册处理器
  this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
  this.remotingExecutor =
    Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
  this.registerProcessor();
  //2.开启两个定时任务，分别用于扫描broker连接是否有效和打印kv配置
  this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
      NamesrvController.this.routeInfoManager.scanNotActiveBroker();
    }
  }, 5, 10, TimeUnit.SECONDS);
  this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
      NamesrvController.this.kvConfigManager.printAllPeriodically();
    }
  }, 1, 10, TimeUnit.MINUTES);
	
  if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
    //3.监控ssl文件，如果变化则重新加载
    fileWatchService = new FileWatchService(
      //...监控文件变化
  }
  return true;
}
```

##### 2.1.1.创建Netty服务端对象，然后注册处理器

##### 2.1.2.开启两个定时任务

一个用于每隔10s扫描一次broker，移除失活状态的broker；

一个用于每隔10min打印一次配置项

##### 2.1.3.监控ssl文件，如果变化则重新加载

#### 2.2.注册一个jvm hook函数

这也是一种常见的优雅停机的方式。注册一个jvm hook函数，然后再jvm进程关闭之前，先将线程池啊，占用的资源文件这些释放掉

#### 2.3.启动controller

代码如下：

```java
public void start() throws Exception {
  //初始化netty服务器配置，然后启动
  this.remotingServer.start();
	//检测配置文件的变更
  if (this.fileWatchService != null) {
    this.fileWatchService.start();
  }
}
```

## 注册路由信息

注册路由信息的动作，由broker发起，namesrv作为服务器端响应。在namesrv中，所有请求的处理入口都在DefaultRequestProcessor.processRequest()方法中。

我们找到处理注册路由信息的具体方法：RouteInfoManager.registerBroker()

方法如下：

```java
public RegisterBrokerResult registerBroker(..) {
  RegisterBrokerResult result = new RegisterBrokerResult();
  //加锁。说明namesrv对路由的注册是串行处理的
  this.lock.writeLock().lockInterruptibly();
  //验证clusterName是否已经注册过
  Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
  if (null == brokerNames) {
    brokerNames = new HashSet<String>();
    this.clusterAddrTable.put(clusterName, brokerNames);
  }
  brokerNames.add(brokerName);

  boolean registerFirst = false;
  //验证brokerName是否已经注册过
  BrokerData brokerData = this.brokerAddrTable.get(brokerName);
  if (null == brokerData) {
    registerFirst = true;
    brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
    this.brokerAddrTable.put(brokerName, brokerData);
  }
  Map<Long, String> brokerAddrsMap = brokerData.getBrokerAddrs();
  Iterator<Entry<Long, String>> it = brokerAddrsMap.entrySet().iterator();
  while (it.hasNext()) {
    //这里的Entry的key是broker的序号，value是broker的ip
    Entry<Long, String> item = it.next();
    //如果brokerAddr已经存在，但是brokerId变了，则把原来的brokerAddr对应的信息删了
    if (null != brokerAddr && brokerAddr.equals(item.getValue()) && brokerId != item.getKey()) {
      it.remove();
    }
  }

  String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
  registerFirst = registerFirst || (null == oldAddr);
	//如果是master
  if (null != topicConfigWrapper && MixAll.MASTER_ID == brokerId) {
    //如果broker的版本号变了或者是第一次注册
    if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())|| registerFirst) {
      ConcurrentMap<String, TopicConfig> tcTable = topicConfigWrapper.getTopicConfigTable();
      if (tcTable != null) {
        for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
          //变更broker对应的QueueData配置信息
          this.createAndUpdateQueueData(brokerName, entry.getValue());
        }
      }
    }
  }
	//填充broker的心跳信息
  BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,new BrokerLiveInfo(System.currentTimeMillis(),
                                                                 topicConfigWrapper.getDataVersion(),channel,haServerAddr));
	//更新filter server的map
  if (filterServerList != null) {
    if (filterServerList.isEmpty()) {
      this.filterServerTable.remove(brokerAddr);
    } else {
      this.filterServerTable.put(brokerAddr, filterServerList);
    }
  }
	//如果此broker不是master，则也会把当前brokerName对应的master的ip信息返回
  if (MixAll.MASTER_ID != brokerId) {
    String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
    if (masterAddr != null) {
      BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
      if (brokerLiveInfo != null) {
        result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
        result.setMasterAddr(masterAddr);
      }
    }
  }
  return result;
}
```

## 删除路由信息

rocketmq中有两种情况会删除路由信息：

* broker挂了，如果namesrv发现一个broker上次心跳信息距离现在超过120s，则会把这个broker的信息删除
* broker正常关闭，会发送一个删除路由信息的请求给namesrv

broker发送取消注册请求的处理：

```java
public void unregisterBroker(String clusterName,String brokerAddr,String brokerName,long brokerId) {
    try {
      //跟注册时一样，要先加锁
      this.lock.writeLock().lockInterruptibly();
      BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.remove(brokerAddr);
			//移除这个broker相关的filterServer
      this.filterServerTable.remove(brokerAddr);

      boolean removeBrokerName = false;
      BrokerData brokerData = this.brokerAddrTable.get(brokerName);
      if (null != brokerData) {
        String addr = brokerData.getBrokerAddrs().remove(brokerId);
        //如果这个broker的brokerName没有别的broker，则把整个brokerName都删掉
        if (brokerData.getBrokerAddrs().isEmpty()) {
          this.brokerAddrTable.remove(brokerName);
          removeBrokerName = true;
        }
      }
      //删掉brokerName
      if (removeBrokerName) {
        Set<String> nameSet = this.clusterAddrTable.get(clusterName);
        if (nameSet != null) {
          boolean removed = nameSet.remove(brokerName);
          if (nameSet.isEmpty()) {
            this.clusterAddrTable.remove(clusterName);
          }
        }
        //如果某个topic除了这个brokerName以外没有别的brokerName，则把topic也删了
        this.removeTopicByBrokerName(brokerName);
      }
    } finally {
      this.lock.writeLock().unlock();
    }
}
```

定时任务删除路由：

具体方法为RouteInfoManager.scanNotActiveBroker()

```java
public void scanNotActiveBroker() {
  //获取broker的心跳map
  Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
  while (it.hasNext()) {
    Entry<String, BrokerLiveInfo> next = it.next();
    long last = next.getValue().getLastUpdateTimestamp();
    //如果一个broker的最后一次注册请求的时间超过了BROKER_CHANNEL_EXPIRED_TIME，默认120s，则删除这个broker的信息
    if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
      RemotingUtil.closeChannel(next.getValue().getChannel());
      it.remove();
      log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
      this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
    }
  }
}
```



## 路由查询

namesrv从不会主动将路由信息的变更推送到producer或者consumer，都是由客户端自己主动发起路由查询来获取变更。当DefaultRequestProcessor接收到GET_ROUTEINTO_BY_TOPIC请求后，就会把RouteInfoManager内的路由信息装配成TopicRouteData返回给客户端

# namesrv如何达到高可用

从namesrv的分析中，可以发现，namesrv服务器之间并没有什么通信，完全是各玩各的，broker启动时，也是向所有的namesrv发送心跳包。namesrv之间并没有主从关系。客服端查询路由信息时，除非namesrv集群的服务器全部挂了，否则就可以正常提供服务。

# 结语

(参考丁威、周继峰<<RocketMQ技术内幕>>。水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

