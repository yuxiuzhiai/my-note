# 概念

# 组件

## CommitLog

Commitlog是rocketmq的消息存储中最为重要的一个组件。首先，它是rocketmq消息持久化的形式，对应着具体的存储消息的磁盘文件。其次，如果每一条消息都需要写磁盘文件然后刷盘，那整个mq的吞吐量必然大打折扣，所以在rocketmq中，Commitlog除了对应着一个磁盘文件，还对应着一个磁盘文件对应的内存映射。

### MappedFile

每个MappedFile就是一个具体的磁盘上的commitlog文件对应的内存映射。还有个MappedFileQueue对象，维护这一个MappedFile的集合，相当于对应着commitlog文件夹

## DefaultMessageStore$ReputMessageService

commitlog的内容并不能直接作为consumer的数据来源，仅仅是broker的持久化消息。要想发送到broker的message可以被consumer消费，还需要这个ReputMessageService来将消息转发，进而生成每个topic，每个MessageQueue对应的messagequeue文件夹下的文件，才可以对消费者可见

## CommitLogDispatcher

负责派发commitlog中的消息。有一下实现类：

* DefaultMessageStore$CommitLogDispatcherBuildConsumeQueue:负责创建consumequeue消费队列文件
* DefaultMessageStore$CommitLogDispatcherBuildIndex:负责创建index索引文件

# 用法



# 实现

## broker持久化消息的实现

主要逻辑在DefaultMessageStore.putMessage()方法：

```java
public PutMessageResult putMessage(MessageExtBrokerInner msg) {
	//省略各种状态判断的代码
  long beginTime = this.getSystemClock().now();
  //核心逻辑
  PutMessageResult result = this.commitLog.putMessage(msg);
  long elapsedTime = this.getSystemClock().now() - beginTime;
  this.storeStatsService.setPutMessageEntireTimeMax(elapsedTime);
	//记录失败
  if (null == result || !result.isOk()) {
    this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
  }
  return result;
}
```

主要的逻辑都在CommitLog的putMessage()方法中，进入：

```java
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
  //设置一些属性
  msg.setStoreTimestamp(System.currentTimeMillis());
  msg.setBodyCRC(UtilAll.crc32(msg.getBody()));
  AppendMessageResult result = null;
  StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();

  String topic = msg.getTopic();
  int queueId = msg.getQueueId();

  final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
  if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
      || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
    //判断是否是延迟消息
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
  
  //省略非核心逻辑
  
  long eclipsedTimeInLock = 0;

  MappedFile unlockMappedFile = null;
  MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();
	//获取自旋锁或者Reentranlock，取决于配置
  putMessageLock.lock();
  try {
    long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
    this.beginTimeInLock = beginLockTimestamp;
    // Here settings are stored timestamp, in order to ensure an orderly
    // global
    msg.setStoreTimestamp(beginLockTimestamp);
		//获取MappedFile
    if (null == mappedFile || mappedFile.isFull()) {
      mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
    }
		//1.往MappedFile追加消息。核心逻辑所在
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

### 1.往MappedFile追加消息

可以看到在CommitLog中，主要的日志写入的实现被委派到MappedFile中实现，方法为MappedFile.appendMessage，进入方法：

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

#### 阿萨德

```java
public AppendMessageResult doAppend(final long fileFromOffset, final ByteBuffer byteBuffer, final int maxBlank,
                                    final MessageExtBrokerInner msgInner) {
  // STORETIMESTAMP + STOREHOSTADDRESS + OFFSET <br>

  // PHY OFFSET
  long wroteOffset = fileFromOffset + byteBuffer.position();

  int sysflag = msgInner.getSysFlag();

  int bornHostLength = (sysflag & MessageSysFlag.BORNHOST_V6_FLAG) == 0 ? 4 + 4 : 16 + 4;
  int storeHostLength = (sysflag & MessageSysFlag.STOREHOSTADDRESS_V6_FLAG) == 0 ? 4 + 4 : 16 + 4;
  ByteBuffer bornHostHolder = ByteBuffer.allocate(bornHostLength);
  ByteBuffer storeHostHolder = ByteBuffer.allocate(storeHostLength);

  this.resetByteBuffer(storeHostHolder, storeHostLength);
  String msgId;
  if ((sysflag & MessageSysFlag.STOREHOSTADDRESS_V6_FLAG) == 0) {
    msgId = MessageDecoder.createMessageId(this.msgIdMemory, msgInner.getStoreHostBytes(storeHostHolder), wroteOffset);
  } else {
    msgId = MessageDecoder.createMessageId(this.msgIdV6Memory, msgInner.getStoreHostBytes(storeHostHolder), wroteOffset);
  }

  // Record ConsumeQueue information
  keyBuilder.setLength(0);
  keyBuilder.append(msgInner.getTopic());
  keyBuilder.append('-');
  keyBuilder.append(msgInner.getQueueId());
  String key = keyBuilder.toString();
  Long queueOffset = CommitLog.this.topicQueueTable.get(key);
  if (null == queueOffset) {
    queueOffset = 0L;
    CommitLog.this.topicQueueTable.put(key, queueOffset);
  }

  // Transaction messages that require special handling
  final int tranType = MessageSysFlag.getTransactionValue(msgInner.getSysFlag());
  switch (tranType) {
      // Prepared and Rollback message is not consumed, will not enter the
      // consumer queuec
    case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
    case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
      queueOffset = 0L;
      break;
    case MessageSysFlag.TRANSACTION_NOT_TYPE:
    case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
    default:
      break;
  }

  /**
             * Serialize message
             */
  final byte[] propertiesData =
    msgInner.getPropertiesString() == null ? null : msgInner.getPropertiesString().getBytes(MessageDecoder.CHARSET_UTF8);

  final int propertiesLength = propertiesData == null ? 0 : propertiesData.length;

  if (propertiesLength > Short.MAX_VALUE) {
    log.warn("putMessage message properties length too long. length={}", propertiesData.length);
    return new AppendMessageResult(AppendMessageStatus.PROPERTIES_SIZE_EXCEEDED);
  }

  final byte[] topicData = msgInner.getTopic().getBytes(MessageDecoder.CHARSET_UTF8);
  final int topicLength = topicData.length;

  final int bodyLength = msgInner.getBody() == null ? 0 : msgInner.getBody().length;

  final int msgLen = calMsgLength(msgInner.getSysFlag(), bodyLength, topicLength, propertiesLength);

  // Exceeds the maximum message
  if (msgLen > this.maxMessageSize) {
    CommitLog.log.warn("message size exceeded, msg total size: " + msgLen + ", msg body size: " + bodyLength
                       + ", maxMessageSize: " + this.maxMessageSize);
    return new AppendMessageResult(AppendMessageStatus.MESSAGE_SIZE_EXCEEDED);
  }

  // Determines whether there is sufficient free space
  if ((msgLen + END_FILE_MIN_BLANK_LENGTH) > maxBlank) {
    this.resetByteBuffer(this.msgStoreItemMemory, maxBlank);
    // 1 TOTALSIZE
    this.msgStoreItemMemory.putInt(maxBlank);
    // 2 MAGICCODE
    this.msgStoreItemMemory.putInt(CommitLog.BLANK_MAGIC_CODE);
    // 3 The remaining space may be any value
    // Here the length of the specially set maxBlank
    final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
    byteBuffer.put(this.msgStoreItemMemory.array(), 0, maxBlank);
    return new AppendMessageResult(AppendMessageStatus.END_OF_FILE, wroteOffset, maxBlank, msgId, msgInner.getStoreTimestamp(),
                                   queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);
  }

  // Initialization of storage space
  this.resetByteBuffer(msgStoreItemMemory, msgLen);
  // 1 TOTALSIZE
  this.msgStoreItemMemory.putInt(msgLen);
  // 2 MAGICCODE
  this.msgStoreItemMemory.putInt(CommitLog.MESSAGE_MAGIC_CODE);
  // 3 BODYCRC
  this.msgStoreItemMemory.putInt(msgInner.getBodyCRC());
  // 4 QUEUEID
  this.msgStoreItemMemory.putInt(msgInner.getQueueId());
  // 5 FLAG
  this.msgStoreItemMemory.putInt(msgInner.getFlag());
  // 6 QUEUEOFFSET
  this.msgStoreItemMemory.putLong(queueOffset);
  // 7 PHYSICALOFFSET
  this.msgStoreItemMemory.putLong(fileFromOffset + byteBuffer.position());
  // 8 SYSFLAG
  this.msgStoreItemMemory.putInt(msgInner.getSysFlag());
  // 9 BORNTIMESTAMP
  this.msgStoreItemMemory.putLong(msgInner.getBornTimestamp());
  // 10 BORNHOST
  this.resetByteBuffer(bornHostHolder, bornHostLength);
  this.msgStoreItemMemory.put(msgInner.getBornHostBytes(bornHostHolder));
  // 11 STORETIMESTAMP
  this.msgStoreItemMemory.putLong(msgInner.getStoreTimestamp());
  // 12 STOREHOSTADDRESS
  this.resetByteBuffer(storeHostHolder, storeHostLength);
  this.msgStoreItemMemory.put(msgInner.getStoreHostBytes(storeHostHolder));
  // 13 RECONSUMETIMES
  this.msgStoreItemMemory.putInt(msgInner.getReconsumeTimes());
  // 14 Prepared Transaction Offset
  this.msgStoreItemMemory.putLong(msgInner.getPreparedTransactionOffset());
  // 15 BODY
  this.msgStoreItemMemory.putInt(bodyLength);
  if (bodyLength > 0)
    this.msgStoreItemMemory.put(msgInner.getBody());
  // 16 TOPIC
  this.msgStoreItemMemory.put((byte) topicLength);
  this.msgStoreItemMemory.put(topicData);
  // 17 PROPERTIES
  this.msgStoreItemMemory.putShort((short) propertiesLength);
  if (propertiesLength > 0)
    this.msgStoreItemMemory.put(propertiesData);

  final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
  // Write messages to the queue buffer
  byteBuffer.put(this.msgStoreItemMemory.array(), 0, msgLen);

  AppendMessageResult result = new AppendMessageResult(AppendMessageStatus.PUT_OK, wroteOffset, msgLen, msgId,
                                                       msgInner.getStoreTimestamp(), queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);

  switch (tranType) {
    case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
    case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
      break;
    case MessageSysFlag.TRANSACTION_NOT_TYPE:
    case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
      // The next update ConsumeQueue information
      CommitLog.this.topicQueueTable.put(key, ++queueOffset);
      break;
    default:
      break;
  }
  return result;
}
```

## commitlog消息的转发

主要的逻辑在DefaultMessageQueue的内部类ReputMessageService中实现。ReputMessageService简洁继承了Runable接口，作为单独的线程，内部的run方法就是每个1ms执行一次doReput()方法。doReput()方法如下：（删减了一下非核心代码）

```java
private void doReput() {
  //获取commitlog中的最小偏移量
  if (this.reputFromOffset < DefaultMessageStore.this.commitLog.getMinOffset()) {
    this.reputFromOffset = DefaultMessageStore.this.commitLog.getMinOffset();
  }
	//返回commitlog中reputFromOffset偏移开始的所有message
  SelectMappedBufferResult result = DefaultMessageStore.this.commitLog.getData(reputFromOffset);
  this.reputFromOffset = result.getStartOffset();
  //遍历每个message
  for (int readSize = 0; readSize < result.getSize() && doNext; ) {
    DispatchRequest dispatchRequest =
      DefaultMessageStore.this.commitLog.checkMessageAndReturnSize(result.getByteBuffer(), false, false);
    int size = dispatchRequest.getBufferSize() == -1 ? dispatchRequest.getMsgSize() : dispatchRequest.getBufferSize();
    if (dispatchRequest.isSuccess()) {
      if (size > 0) {
        //消息的转发
        DefaultMessageStore.this.doDispatch(dispatchRequest);
        if (BrokerRole.SLAVE != DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole()
            && DefaultMessageStore.this.brokerConfig.isLongPollingEnable()) {
          DefaultMessageStore.this.messageArrivingListener.arriving(dispatchRequest.getTopic(),
                                                                    dispatchRequest.getQueueId(), dispatchRequest.getConsumeQueueOffset() + 1,
                                                                    dispatchRequest.getTagsCode(), dispatchRequest.getStoreTimestamp(),
                                                                    dispatchRequest.getBitMap(), dispatchRequest.getPropertiesMap());
        }

        this.reputFromOffset += size;
        readSize += size;
        if (DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole() == BrokerRole.SLAVE) {
          DefaultMessageStore.this.storeStatsService
            .getSinglePutMessageTopicTimesTotal(dispatchRequest.getTopic()).incrementAndGet();
          DefaultMessageStore.this.storeStatsService
            .getSinglePutMessageTopicSizeTotal(dispatchRequest.getTopic())
            .addAndGet(dispatchRequest.getMsgSize());
        }
      } else if (size == 0) {
        this.reputFromOffset = DefaultMessageStore.this.commitLog.rollNextFile(this.reputFromOffset);
        readSize = result.getSize();
      }
    } else if (!dispatchRequest.isSuccess()) {
      if (size > 0) {
        this.reputFromOffset += size;
      } else {
        doNext = false;
        // If user open the dledger pattern or the broker is master node,
        // it will not ignore the exception and fix the reputFromOffset variable
        if (DefaultMessageStore.this.getMessageStoreConfig().isEnableDLegerCommitLog() ||
            DefaultMessageStore.this.brokerConfig.getBrokerId() == MixAll.MASTER_ID) {
          this.reputFromOffset += result.getSize() - readSize;
        }
      }
    }
  }
}
```





# 结语



(水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

