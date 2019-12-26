# 概念

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

# 用法



# 实现

## 生产者的启动

在创建好Producer之后，使用它来发消息之前，需要先启动它，即调用它的start()方法，代码如下：

```java
public void start() throws MQClientException {
  this.setProducerGroup(withNamespace(this.producerGroup));
  //1.1.DefaultMQProducerImpl的启动
  this.defaultMQProducerImpl.start();
  if (null != traceDispatcher) {
    try {
      //1.2.启动TraceDispatcher
      traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
    } catch (MQClientException e) {
      log.warn("trace dispatcher start failed ", e);
    }
  }
}
```

### 1.1.DefaultMQProducerImpl的启动

代码如下

```java
public void start(final boolean startFactory) throws MQClientException {
  //这里的switch主要是用来确保start()方法必须在使用前被调用并且只能调用一次
  switch (this.serviceState) {
    case CREATE_JUST:
      this.serviceState = ServiceState.START_FAILED;

      this.checkConfig();

      if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
        this.defaultMQProducer.changeInstanceNameToPID();
      }
			//创建一个MQClientInstance实例
      this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);
			//注册到MQClientInstance
      boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
      if (!registerOK) {
        this.serviceState = ServiceState.CREATE_JUST;
        throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                                    + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                                    null);
      }

      this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

      if (startFactory) {
        //启动MQClientInstance
        mQClientFactory.start();
      }

      log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
               this.defaultMQProducer.isSendMessageWithVIPChannel());
      this.serviceState = ServiceState.RUNNING;
      break;
    case RUNNING:
    case START_FAILED:
    case SHUTDOWN_ALREADY:
      throw new MQClientException("The producer service state not OK, maybe started once, "
                                  + this.serviceState
                                  + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                                  null);
    default:
      break;
  }
	//发送心跳消息
  this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
	//启动线程定时清理超时的请求	
  this.timer.scheduleAtFixedRate(new TimerTask() {
    @Override
    public void run() {
      try {
        RequestFutureTable.scanExpiredRequest();
      } catch (Throwable e) {
        log.error("scan RequestFutureTable exception", e);
      }
    }
  }, 1000 * 3, 1000);
}
```

### 1.2.TraceDispatcher的启动

## 2.发送消息

主要实现在DefaultMQProducerImpl.sendDefaultImpl()方法中。进入方法

```java
private SendResult sendDefaultImpl(Message msg,final CommunicationMode communicationMode,final SendCallback sendCallback,
                                   final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
  this.makeSureStateOK();
  Validators.checkMessage(msg, this.defaultMQProducer);
  final long invokeID = random.nextLong();
  long beginTimestampFirst = System.currentTimeMillis();
  long beginTimestampPrev = beginTimestampFirst;
  long endTimestamp = beginTimestampFirst;
  //2.1.查找主题的路由信息
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
      //2.2.选择消息队列
      MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
      if (mqSelected != null) {
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
					//2.3.发送消息
          sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
          endTimestamp = System.currentTimeMillis();
          this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
          switch (communicationMode) {
            case ASYNC:
              return null;
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
        } catch (RemotingException e) {
          endTimestamp = System.currentTimeMillis();
          this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
          log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
          log.warn(msg.toString());
          exception = e;
          continue;
        } catch (MQClientException e) {
          endTimestamp = System.currentTimeMillis();
          this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
          log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
          log.warn(msg.toString());
          exception = e;
          continue;
        } catch (MQBrokerException e) {
          endTimestamp = System.currentTimeMillis();
          this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
          log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
          log.warn(msg.toString());
          exception = e;
          switch (e.getResponseCode()) {
            case ResponseCode.TOPIC_NOT_EXIST:
            case ResponseCode.SERVICE_NOT_AVAILABLE:
            case ResponseCode.SYSTEM_ERROR:
            case ResponseCode.NO_PERMISSION:
            case ResponseCode.NO_BUYER_ID:
            case ResponseCode.NOT_IN_CURRENT_UNIT:
              continue;
            default:
              if (sendResult != null) {
                return sendResult;
              }

              throw e;
          }
        } catch (InterruptedException e) {
          endTimestamp = System.currentTimeMillis();
          this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
          log.warn(String.format("sendKernelImpl exception, throw exception, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
          log.warn(msg.toString());

          log.warn("sendKernelImpl exception", e);
          log.warn(msg.toString());
          throw e;
        }
      } else {
        break;
      }
    }

    if (sendResult != null) {
      return sendResult;
    }

    String info = String.format("Send [%d] times, still failed, cost [%d]ms, Topic: %s, BrokersSent: %s",
                                times,
                                System.currentTimeMillis() - beginTimestampFirst,
                                msg.getTopic(),
                                Arrays.toString(brokersSent));

    info += FAQUrl.suggestTodo(FAQUrl.SEND_MSG_FAILED);

    MQClientException mqClientException = new MQClientException(info, exception);
    if (callTimeout) {
      throw new RemotingTooMuchRequestException("sendDefaultImpl call timeout");
    }

    if (exception instanceof MQBrokerException) {
      mqClientException.setResponseCode(((MQBrokerException) exception).getResponseCode());
    } else if (exception instanceof RemotingConnectException) {
      mqClientException.setResponseCode(ClientErrorCode.CONNECT_BROKER_EXCEPTION);
    } else if (exception instanceof RemotingTimeoutException) {
      mqClientException.setResponseCode(ClientErrorCode.ACCESS_BROKER_TIMEOUT);
    } else if (exception instanceof MQClientException) {
      mqClientException.setResponseCode(ClientErrorCode.BROKER_NOT_EXIST_EXCEPTION);
    }

    throw mqClientException;
  }

  validateNameServerSetting();

  throw new MQClientException("No route info of this topic: " + msg.getTopic() + FAQUrl.suggestTodo(FAQUrl.NO_TOPIC_ROUTE_INFO),
                              null).setResponseCode(ClientErrorCode.NOT_FOUND_TOPIC_EXCEPTION);
}
```

### 2.1.查找主题的路由信息

```java
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
  TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
  if (null == topicPublishInfo || !topicPublishInfo.ok()) {
    this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
    this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
    topicPublishInfo = this.topicPublishInfoTable.get(topic);
  }

  if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
    return topicPublishInfo;
  } else {
    this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
    topicPublishInfo = this.topicPublishInfoTable.get(topic);
    return topicPublishInfo;
  }
}
```



### 2.2.选择消息队列

在rocketmq中，选择消息队列，大体上有两种方案。根据sendLatencyFaultEnable(故障延迟机制)属性值来判断（默认为false）。如果sendLatencyFaultEnable = true：

#### 2.2.1. 开启了故障延迟机制下的MessageQueue选择

```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
  if (this.sendLatencyFaultEnable) {
    //对应这一段代码
    try {
      //
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
    } catch (Exception e) {
      log.error("Error occurred when selecting message queue", e);
    }

    return tpInfo.selectOneMessageQueue();
  }
	//见2.2.2的分析
  return tpInfo.selectOneMessageQueue(lastBrokerName);
}
```



#### 2.2.2. 默认机制(未开启故障延迟机制)

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

### 2.3.发送消息

发送消息的入口方法：DefaultMQProducerImpl.sendKernelImpl()

方法行数比较多，去掉了try catch和一些不重要的部分

```java
private SendResult sendKernelImpl(final Message msg,final MessageQueue mq,final CommunicationMode communicationMode,
                                  final SendCallback sendCallback,final TopicPublishInfo topicPublishInfo,final long timeout){
  long beginStartTime = System.currentTimeMillis();
  String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
  //根据topic的路由信息拿到broker的地址
  if (null == brokerAddr) {
    tryToFindTopicPublishInfo(mq.getTopic());
    brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
  }

  SendMessageContext context = null;
  if (brokerAddr != null) {
    brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);

    byte[] prevBody = msg.getBody();

    //for MessageBatch,ID has been set in the generating process
    if (!(msg instanceof MessageBatch)) {
      //给消息加上一个属性UNIQ_KEY
      MessageClientIDSetter.setUniqID(msg);
    }

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

      if (msg.getProperty("__STARTDELIVERTIME") != null || msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null) {
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
      //异步发送消息
      case ASYNC:
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
        sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
          brokerAddr,
          mq.getBrokerName(),
          tmpMessage,
          requestHeader,
          timeout - costTimeAsync,
          communicationMode,
          sendCallback,
          topicPublishInfo,
          this.mQClientFactory,
          this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(),
          context,
          this);
        break;
      //同步或者oneway发送消息
      case ONEWAY:
      case SYNC:
        long costTimeSync = System.currentTimeMillis() - beginStartTime;
        if (timeout < costTimeSync) {
          throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
        }
        sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
          brokerAddr,
          mq.getBrokerName(),
          msg,
          requestHeader,
          timeout - costTimeSync,
          communicationMode,
          context,
          this);
        break;
      default:
        assert false;
        break;
    }

    if (this.hasSendMessageHook()) {
      context.setSendResult(sendResult);
      this.executeSendMessageHookAfter(context);
    }
    return sendResult;
  }

  throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
}
```



# 结语



(水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

