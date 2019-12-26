# 概念

# 组件

## CommitLog

### MappedFile



# 用法



# 实现

```java
public PutMessageResult putMessage(MessageExtBrokerInner msg) {
	//省略各种状态判断的代码
  
  long beginTime = this.getSystemClock().now();
  //核心逻辑
  PutMessageResult result = this.commitLog.putMessage(msg);

  long elapsedTime = this.getSystemClock().now() - beginTime;
  if (elapsedTime > 500) {
    log.warn("putMessage not in lock elapsed time(ms)={}, bodyLength={}", elapsedTime, msg.getBody().length);
  }
  this.storeStatsService.setPutMessageEntireTimeMax(elapsedTime);

  if (null == result || !result.isOk()) {
    this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
  }

  return result;
}
```

主要的逻辑都在CommitLog的putMessage()方法中，进入：

```java
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
  // Set the storage time
  msg.setStoreTimestamp(System.currentTimeMillis());
  // Set the message body BODY CRC (consider the most appropriate setting
  // on the client)
  msg.setBodyCRC(UtilAll.crc32(msg.getBody()));
  // Back to Results
  AppendMessageResult result = null;

  StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();

  String topic = msg.getTopic();
  int queueId = msg.getQueueId();

  final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
  if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
      || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
    // Delay Delivery
    if (msg.getDelayTimeLevel() > 0) {
      if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
        msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
      }

      topic = ScheduleMessageService.SCHEDULE_TOPIC;
      queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

      // Backup real topic, queueId
      MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
      MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
      msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));

      msg.setTopic(topic);
      msg.setQueueId(queueId);
    }
  }

  InetSocketAddress bornSocketAddress = (InetSocketAddress) msg.getBornHost();
  if (bornSocketAddress.getAddress() instanceof Inet6Address) {
    msg.setBornHostV6Flag();
  }

  InetSocketAddress storeSocketAddress = (InetSocketAddress) msg.getStoreHost();
  if (storeSocketAddress.getAddress() instanceof Inet6Address) {
    msg.setStoreHostAddressV6Flag();
  }

  long eclipsedTimeInLock = 0;

  MappedFile unlockMappedFile = null;
  MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();

  putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
  try {
    long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
    this.beginTimeInLock = beginLockTimestamp;

    // Here settings are stored timestamp, in order to ensure an orderly
    // global
    msg.setStoreTimestamp(beginLockTimestamp);

    if (null == mappedFile || mappedFile.isFull()) {
      mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
    }
    if (null == mappedFile) {
      log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
      beginTimeInLock = 0;
      return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
    }
		//主要逻辑又走到了MappedFile
    result = mappedFile.appendMessage(msg, this.appendMessageCallback);
    switch (result.getStatus()) {
      case PUT_OK:
        break;
      case END_OF_FILE:
        unlockMappedFile = mappedFile;
        // Create a new file, re-write the message
        mappedFile = this.mappedFileQueue.getLastMappedFile(0);
        if (null == mappedFile) {
          // XXX: warn and notify me
          log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
          beginTimeInLock = 0;
          return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
        }
        result = mappedFile.appendMessage(msg, this.appendMessageCallback);
        break;
      case MESSAGE_SIZE_EXCEEDED:
      case PROPERTIES_SIZE_EXCEEDED:
        beginTimeInLock = 0;
        return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);
      case UNKNOWN_ERROR:
        beginTimeInLock = 0;
        return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
      default:
        beginTimeInLock = 0;
        return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
    }

    eclipsedTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
    beginTimeInLock = 0;
  } finally {
    putMessageLock.unlock();
  }

  if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
    this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
  }

  PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

  // Statistics
  storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).incrementAndGet();
  storeStatsService.getSinglePutMessageTopicSizeTotal(topic).addAndGet(result.getWroteBytes());
	//刷盘处理
  handleDiskFlush(result, putMessageResult, msg);
  //HA处理
  handleHA(result, putMessageResult, msg);

  return putMessageResult;
}
```

## MappedFile.appendMessage的实现

可以看到在CommitLog中，主要的日志写入的实现被委派到MappedFile中实现，进入方法：

```java
public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
 
  int currentPos = this.wrotePosition.get();

  if (currentPos < this.fileSize) {
    ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
    byteBuffer.position(currentPos);
    AppendMessageResult result;
    if (messageExt instanceof MessageExtBrokerInner) {
      //序列化并存储消息
      result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);
    } else if (messageExt instanceof MessageExtBatch) {
      result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch) messageExt);
    } else {
      return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
    }
    this.wrotePosition.addAndGet(result.getWroteBytes());
    this.storeTimestamp = result.getStoreTimestamp();
    return result;
  }
  return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
}
```



# 结语



(水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

