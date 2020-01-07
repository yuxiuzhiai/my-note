# 概念

生产者producer，用于生产消息，在rocketmq中对应着MQProducer接口。

# 组件

## Producer

消息生产者。在rocketmq中，生产者对应MQProducer接口：

```java
public interface MQProducer extends MQAdmin {
  //启动
  void start() throws MQClientException;
	//关闭
  void shutdown();
	
  List<MessageQueue> fetchPublishMessageQueues(final String topic) throws MQClientException;
	//同步发送消息
  SendResult send(final Message msg) throws MQClientException, RemotingException, MQBrokerException,
  InterruptedException;
	//异步发送消息
  void send(final Message msg, final SendCallback sendCallback) throws MQClientException,
  RemotingException, InterruptedException;
	//oneway形式发送消息，相较于异步发送，其实就是没有注册回调函数
  void sendOneway(final Message msg) throws MQClientException, RemotingException,
  InterruptedException;
	//发送事务消息
  TransactionSendResult sendMessageInTransaction(final Message msg,
                                                 final Object arg) throws MQClientException;
  //批量发送消息
  SendResult send(final Collection<Message> msgs) throws MQClientException, RemotingException, MQBrokerException,
  InterruptedException;
  
  //还有各种形式的重载的发送消息的方法，省略了。。。

  //for rpc
  Message request(final Message msg, final long timeout) throws RequestTimeoutException, MQClientException,
  RemotingException, MQBrokerException, InterruptedException;

  void request(final Message msg, final RequestCallback requestCallback, final long timeout)
    throws MQClientException, RemotingException, InterruptedException, MQBrokerException;

  Message request(final Message msg, final MessageQueueSelector selector, final Object arg,
                  final long timeout) throws RequestTimeoutException, MQClientException, RemotingException, MQBrokerException,
  InterruptedException;

  void request(final Message msg, final MessageQueueSelector selector, final Object arg,
               final RequestCallback requestCallback,
               final long timeout) throws MQClientException, RemotingException,
  InterruptedException, MQBrokerException;

  Message request(final Message msg, final MessageQueue mq, final long timeout)
    throws RequestTimeoutException, MQClientException, RemotingException, MQBrokerException, InterruptedException;

  void request(final Message msg, final MessageQueue mq, final RequestCallback requestCallback, long timeout)
    throws MQClientException, RemotingException, MQBrokerException, InterruptedException;
}
```

有两个实现：

* DefaultMQProducer：默认实现。没有实现发送事务消息的方法
* TransactionMQProducer：继承自DefaultMQProducer，实现了发送事务消息的方法

## Message

在rocketmq中，Message类就代表着生产者产出的消息。一次看看它的属性：

* topic：主题

* flag：一些特殊的消息标记，int类型。标记的含义定义在MessageSysFlag中：

  ```java
  public final static int COMPRESSED_FLAG = 0x1;
  public final static int MULTI_TAGS_FLAG = 0x1 << 1;
  public final static int TRANSACTION_NOT_TYPE = 0;
  public final static int TRANSACTION_PREPARED_TYPE = 0x1 << 2;
  public final static int TRANSACTION_COMMIT_TYPE = 0x2 << 2;
  public final static int TRANSACTION_ROLLBACK_TYPE = 0x3 << 2;
  public final static int BORNHOST_V6_FLAG = 0x1 << 4;
  public final static int STOREHOSTADDRESS_V6_FLAG = 0x1 << 5;
  ```

  

* properties：额外的一些属性，Map类型。已经使用的扩展属性有：

  * tags:消息过滤用
  * keys：索引，可以有多个
  * waitStoreMsgOK
  * delayTimeLevel：消息延迟级别，用于定时消息或者消息重试

* body：消息的内容

* transactionId：事务消息用

## MQClientInstance

见：[MQClientInstance](https://blog.csdn.net/yuxiuzhiai/article/details/103828284)

## MQFaultStrategy

## SendMessageHook

发送消息时的hook函数，可以再消息发送前后做一些业务操作。接口定义如下：

```java
public interface SendMessageHook {
  //命名
  String hookName();
	//发送消息前调用
  void sendMessageBefore(final SendMessageContext context);
	//发送消息后调用
  void sendMessageAfter(final SendMessageContext context);
}
```



# 使用



# 实现

本篇文章，我们会逐个分析下列过程：

* 生产者的启动
* 发送消息

(代码有删减，去除了try catch和日志等不太要紧的部分)

## 生产者的启动

在创建好Producer之后，使用它来发消息之前，需要先启动它，即调用它的start()方法，代码如下：

```java
public void start() throws MQClientException {
  this.setProducerGroup(withNamespace(this.producerGroup));
  //DefaultMQProducerImpl的启动
  this.defaultMQProducerImpl.start();
  if (null != traceDispatcher) {
    //TraceDispatcher相关见：
    traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
  }
}
```

本文主要看DefaultMQProducerImpl的启动，代码如下

```java
public void start(final boolean startFactory) throws MQClientException {
  //一些配置校验
  this.checkConfig();
  //如果没有特别指定producerGroup，就会把instanceName设置为进程id
  if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
    this.defaultMQProducer.changeInstanceNameToPID();
  }
  //创建一个MQClientInstance实例
  this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);
  //注册到MQClientInstance
  boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
  this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());
  if (startFactory) {
    //启动MQClientInstance,见[MQClientInstance](https://blog.csdn.net/yuxiuzhiai/article/details/103828284)
    mQClientFactory.start();
  }
  //发送心跳消息给broker
  this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
  //启动线程定时清理超时的请求	
  this.timer.scheduleAtFixedRate(new TimerTask() {
    @Override
    public void run() {
      RequestFutureTable.scanExpiredRequest();
    }
  }, 1000 * 3, 1000);
}
```

## 发送消息

已最普通的同步消息发送为例。主要实现在DefaultMQProducerImpl.sendDefaultImpl()方法中。进入方法

```java
private SendResult sendDefaultImpl(Message msg, CommunicationMode communicationMode, SendCallback sendCallback,long timeout){
  final long invokeID = random.nextLong();
  long beginTimestampFirst = System.currentTimeMillis();
  long beginTimestampPrev = beginTimestampFirst;
  long endTimestamp = beginTimestampFirst;
  //1.查找主题的路由信息
  TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
  if (topicPublishInfo != null && topicPublishInfo.ok()) {
    boolean callTimeout = false;
    MessageQueue mq = null;
    Exception exception = null;
    SendResult sendResult = null;
    //最大可能的发送次数，同步的话就是重试次数+1，异步或者oneway就是只发一次
    int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
    int times = 0;
    String[] brokersSent = new String[timesTotal];
    //重试机制
    for (; times < timesTotal; times++) {
      String lastBrokerName = null == mq ? null : mq.getBrokerName();
      //2.选择消息队列
      MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
      mq = mqSelected;
      brokersSent[times] = mq.getBrokerName();
      try {
        beginTimestampPrev = System.currentTimeMillis();
        if (times > 0) {
          //Reset topic with namespace during resend.
          msg.setTopic(this.defaultMQProducer.withNamespace(msg.getTopic()));
        }
        long costTime = beginTimestampPrev - beginTimestampFirst;
        if (timeout < costTime) {
          callTimeout = true;
          break;
        }
        //3.发送消息
        sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
        endTimestamp = System.currentTimeMillis();
        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
        switch (communicationMode) {
          case ASYNC:
          case ONEWAY:
            return null;
          case SYNC:
            if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
              if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                continue;
              }
            }
            return sendResult;
          default:
            break;
        }
      } 
      //4.异常处理机制
      catch (各种异常 e) {
        endTimestamp = System.currentTimeMillis();
        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
        exception = e;
        continue;
      }
    }
    if (sendResult != null) {
      return sendResult;
    }
    //如果sendResult是null，则说明有异常，进行异常处理
  }
}
```

### 1.查找主题的路由信息

如果是第一次，则会从namesrv获取topic元数据，获取后会缓存下来，以后从缓存中获取

```java
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
  //尝试从缓存获取
  TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
  //如果本地缓存没有或者有问题，则从namesrv获取
  if (null == topicPublishInfo || !topicPublishInfo.ok()) {
    this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
    this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
    topicPublishInfo = this.topicPublishInfoTable.get(topic);
  }
  //获取到了，返回
  if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
    return topicPublishInfo;
  } else {
		//默认topic：TBW102
    this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
    topicPublishInfo = this.topicPublishInfoTable.get(topic);
    return topicPublishInfo;
  }
}
```



### 2.选择消息队列

在rocketmq中，选择消息队列，大体上有两种方案。根据sendLatencyFaultEnable(故障延迟机制)属性值来判断（默认为false）。如果sendLatencyFaultEnable = true：

#### 2.1. 开启了故障延迟机制下的MessageQueue选择

```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
  if (this.sendLatencyFaultEnable) {
    //对应这一段代码
    //大概意思就是根据MessageQueue的数量，round-robin负责均衡
    int index = tpInfo.getSendWhichQueue().getAndIncrement();
    for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
      int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
      if (pos < 0)
        pos = 0;
      MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
      if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
        if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
          return mq;
      }
    }
		//如果正常的话，是走不到这里的。走到这里说明故障延迟机制下没有可用的brokerName
    //这个时候就强行挑一个发送
    final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
    int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
    if (writeQueueNums > 0) {
      final MessageQueue mq = tpInfo.selectOneMessageQueue();
      if (notBestBroker != null) {
        mq.setBrokerName(notBestBroker);
        mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
      }
      return mq;
    } else {
      latencyFaultTolerance.remove(notBestBroker);
    }
    return tpInfo.selectOneMessageQueue();
  }
  //见2.2.2的分析
  return tpInfo.selectOneMessageQueue(lastBrokerName);
}
```

故障延迟机制见[故障延迟机制](https://blog.csdn.net/yuxiuzhiai/article/details/103740627)

#### 2.2. 默认机制(未开启故障延迟机制)

```java
public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
  //这条消息是否是首次发送，如果不是的话，则lastBrokerName就是上次发送失败的broker的name
  if (lastBrokerName == null) {
    //如果这条消息是第一次发送，则直接用一种round-robin方式挑选MessageQueue
    return selectOneMessageQueue();
  } else {
    //如果消息不是第一次发送，则本次挑选MessageQueue的时候尽量避免上次失败的broker
    int index = this.sendWhichQueue.getAndIncrement();
    for (int i = 0; i < this.messageQueueList.size(); i++) {
      int pos = Math.abs(index++) % this.messageQueueList.size();
      if (pos < 0)
        pos = 0;
      MessageQueue mq = this.messageQueueList.get(pos);
      if (!mq.getBrokerName().equals(lastBrokerName)) {
        return mq;
      }
    }
    //如果就一个broker，那避免不了，就硬着头皮随机选一个
    return selectOneMessageQueue();
  }
}
```

### 3.发送消息

发送消息的入口方法：DefaultMQProducerImpl.sendKernelImpl()

```java
private SendResult sendKernelImpl( Message msg, MessageQueue mq, CommunicationMode communicationMode,
                                  SendCallback sendCallback, TopicPublishInfo topicPublishInfo, long timeout){
  long beginStartTime = System.currentTimeMillis();
  //根据topic的路由信息拿到broker的地址
  String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
  if (null == brokerAddr) {
    tryToFindTopicPublishInfo(mq.getTopic());
    brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
  }
  SendMessageContext context = null;
  //有个是否启用vip通道的配置项，如果开启了，则会使用broker的另一个端口发送消息
  brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);
  byte[] prevBody = msg.getBody();
  //设置一个唯一的id
  if (!(msg instanceof MessageBatch)) {
    //给消息加上一个属性UNIQ_KEY
    MessageClientIDSetter.setUniqID(msg);
  }
	//namespace，还没搞懂
  boolean topicWithNamespace = false;
  if (null != this.mQClientFactory.getClientConfig().getNamespace()) {
    msg.setInstanceId(this.mQClientFactory.getClientConfig().getNamespace());
    topicWithNamespace = true;
  }

  int sysFlag = 0;
  boolean msgBodyCompressed = false;
  //根据消息长度看看是不是要压缩一下
  if (this.tryToCompressMessage(msg)) {
    sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
    msgBodyCompressed = true;
  }
  //判断是否是事务消息
  final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
  if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
    sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
  }

  if (hasCheckForbiddenHook()) {
    CheckForbiddenContext checkForbiddenContext = new CheckForbiddenContext();
    //设置各种属性
    checkForbiddenContext.setNameSrvAddr/Group/CommunicationMode/BrokerAddr/Message/Mq/UnitMode..();
    this.executeCheckForbiddenHook(checkForbiddenContext);
  }
  //SendMessageHook是一个用于自定义在消息发送前后做一些自定义处理的接口
  if (this.hasSendMessageHook()) {
    context = new SendMessageContext();
    //设置各种属性
    context.setProducer/ProducerGroup/CommunicationMode ..();
    String isTrans = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
    if (isTrans != null && isTrans.equals("true")) {
      context.setMsgType(MessageType.Trans_Msg_Half);
    }
		//判断是否是延时消息
    if (msg.getProperty("__STARTDELIVERTIME") != null ||msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null){
      context.setMsgType(MessageType.Delay_Msg);
    }
    this.executeSendMessageHookBefore(context);
  }
  //构建SendMessageRequestHeader
  SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
  //省略设置各种属性的代码
  if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
    String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);
    if (reconsumeTimes != null) {
      requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
      MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
    }
    String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
    if (maxReconsumeTimes != null) {
      requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
      MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
    }
  }

  SendResult sendResult = null;
  switch (communicationMode) {
    case ASYNC://异步发送消息
      Message tmpMessage = msg;
      boolean messageCloned = false;
      if (msgBodyCompressed) {
        tmpMessage = MessageAccessor.cloneMessage(msg);
        messageCloned = true;
        msg.setBody(prevBody);
      }

      if (topicWithNamespace) {
        if (!messageCloned) {
          tmpMessage = MessageAccessor.cloneMessage(msg);
          messageCloned = true;
        }
        msg.setTopic(NamespaceUtil.withoutNamespace(msg.getTopic(), this.defaultMQProducer.getNamespace()));
      }

      long costTimeAsync = System.currentTimeMillis() - beginStartTime;
      if (timeout < costTimeAsync) {
        throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
      }
      sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(..);
      break;
    case ONEWAY:
    case SYNC://同步或者oneway发送消息
      long costTimeSync = System.currentTimeMillis() - beginStartTime;
      if (timeout < costTimeSync) {
        throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
      }
      sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(..);
      break;
    default:
      assert false;
      break;
  }
  //SendMessageHook发送消息后的回调
  if (this.hasSendMessageHook()) {
    context.setSendResult(sendResult);
    this.executeSendMessageHookAfter(context);
  }
  return sendResult;
}
```

### 4.异常处理机制

可以看到，当发送消息出现异常时，都有一句：this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false)。具体分析请看[故障延迟机制](https://blog.csdn.net/yuxiuzhiai/article/details/103740627)

## 批量发送消息

DefaultMQProducer.send(Collection<Message> msgs)方法定义如下：

```java
public SendResult send(Collection<Message> msgs){
  //主要逻辑就是batch()方法
  return this.defaultMQProducerImpl.send(batch(msgs));
}
```

batch()方法的实现：

```java
private MessageBatch batch(Collection<Message> msgs) throws MQClientException {
  //将批量的消息转化为一个MessageBatch对象
  MessageBatch msgBatc = MessageBatch.generateFromList(msgs);
  for (Message message : msgBatch) {
    //为每一条单独的message设置uniq key、topic
    Validators.checkMessage(message, this);
    MessageClientIDSetter.setUniqID(message);
    message.setTopic(withNamespace(message.getTopic()));
  }
  //批量消息跟普通消息的发送没有啥差别，只是将消息序列化成字节数组的时候有点不一样
  msgBatch.setBody(msgBatch.encode());
  msgBatch.setTopic(withNamespace(msgBatch.getTopic()));
  return msgBatch;
}
```

序列化MessageBatch的过程：

```java
public static byte[] encodeMessages(List<Message> messages) {
  //TO DO refactor, accumulate in one buffer, avoid copies
  List<byte[]> encodedMessages = new ArrayList<byte[]>(messages.size());
  int allSize = 0;
  for (Message message : messages) {
    //按照固定的格式序列化每一条message
    byte[] tmp = encodeMessage(message);
    encodedMessages.add(tmp);
    allSize += tmp.length;
  }
  byte[] allBytes = new byte[allSize];
  int pos = 0;
  for (byte[] bytes : encodedMessages) {
    System.arraycopy(bytes, 0, allBytes, pos, bytes.length);
    pos += bytes.length;
  }
  return allBytes;
}
```

单独一条消息的序列化过程：

```java
public static byte[] encodeMessage(Message message) {
  byte[] body = message.getBody();
  int bodyLen = body.length;
  String properties = messageProperties2String(message.getProperties());
  byte[] propertiesBytes = properties.getBytes(CHARSET_UTF8);
  //note properties length must not more than Short.MAX
  short propertiesLength = (short) propertiesBytes.length;
  int sysFlag = message.getFlag();
  int storeSize = 4 // 1 TOTALSIZE
    + 4 // 2 MAGICCOD
    + 4 // 3 BODYCRC
    + 4 // 4 FLAG
    + 4 + bodyLen // 4 BODY
    + 2 + propertiesLength;
  ByteBuffer byteBuffer = ByteBuffer.allocate(storeSize);
  // 1 TOTALSIZE
  byteBuffer.putInt(storeSize);
  // 2 MAGICCODE
  byteBuffer.putInt(0);
  // 3 BODYCRC
  byteBuffer.putInt(0);
  // 4 FLAG
  int flag = message.getFlag();
  byteBuffer.putInt(flag);
  // 5 BODY
  byteBuffer.putInt(bodyLen);
  byteBuffer.put(body);
  // 6 properties
  byteBuffer.putShort(propertiesLength);
  byteBuffer.put(propertiesBytes);
  return byteBuffer.array();
}
```

# 结语

(参考丁威、周继峰<<RocketMQ技术内幕>>。水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)