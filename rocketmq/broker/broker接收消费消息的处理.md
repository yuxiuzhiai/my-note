# 概念

# 组件

## SubscriptionGroupManager

消费者组的订阅信息管理器。内部有一个 消费者组 -> 订阅信息的map

## ConsumerManager

消费者组管理器。内部有一个消费者组 -> ConsumerGroupInfo的map，而ConsumerGroupInfo内部又有一个topic -> SubscriptionData的map

# 用法



# 实现

消费者拉取消息的请求code是PULL_MESSAGE。找到对应的处理方法：PullMessageProcessor.processRequest。进入方法：

```java
private RemotingCommand processRequest(final Channel channel, RemotingCommand request, boolean brokerAllowSuspend)
  throws RemotingCommandException {
  RemotingCommand response = RemotingCommand.createResponseCommand(PullMessageResponseHeader.class);
  final PullMessageResponseHeader responseHeader = (PullMessageResponseHeader) response.readCustomHeader();
  final PullMessageRequestHeader requestHeader =
    (PullMessageRequestHeader) request.decodeCommandCustomHeader(PullMessageRequestHeader.class);

  response.setOpaque(request.getOpaque());
  //1.获取消费者组订阅信息
  SubscriptionGroupConfig subscriptionGroupConfig = ..;
  //解析PullSysFlag
  final boolean hasSuspendFlag = PullSysFlag.hasSuspendFlag(requestHeader.getSysFlag());
  final boolean hasCommitOffsetFlag = PullSysFlag.hasCommitOffsetFlag(requestHeader.getSysFlag());
  final boolean hasSubscriptionFlag = PullSysFlag.hasSubscriptionFlag(requestHeader.getSysFlag());

  final long suspendTimeoutMillisLong = hasSuspendFlag ? requestHeader.getSuspendTimeoutMillis() : 0;
  //获取topic配置信息
  TopicConfig topicConfig = ..;
  SubscriptionData subscriptionData = null;
  ConsumerFilterData consumerFilterData = null;

  if (hasSubscriptionFlag) {
    //如果定义了消息过滤器
    subscriptionData = FilterAPI.build(requestHeader.getTopic(), requestHeader.getSubscription(), 						   requestHeader.getExpressionType());
    if (!ExpressionType.isTagType(subscriptionData.getExpressionType())) {
      consumerFilterData = ConsumerFilterManager.build(
        requestHeader.getTopic(), requestHeader.getConsumerGroup(), requestHeader.getSubscription(),
        requestHeader.getExpressionType(), requestHeader.getSubVersion());
    }
  } else {
    //获取消费者组信息
    ConsumerGroupInfo consumerGroupInfo = ..;
		//从消费者组的信息中提取出特定topic的订阅信息
    subscriptionData = ..;
  }

  MessageFilter messageFilter;
  if (this.brokerController.getBrokerConfig().isFilterSupportRetry()) {
    messageFilter = new ExpressionForRetryMessageFilter(subscriptionData, consumerFilterData,
                                                        this.brokerController.getConsumerFilterManager());
  } else {
    messageFilter = new ExpressionMessageFilter(subscriptionData, consumerFilterData,
                                                this.brokerController.getConsumerFilterManager());
  }
	//从DefaultMessageStore中获取消息
  final GetMessageResult getMessageResult =
    this.brokerController.getMessageStore().getMessage(requestHeader.getConsumerGroup(), requestHeader.getTopic(),
                                                       requestHeader.getQueueId(), requestHeader.getQueueOffset(), requestHeader.getMaxMsgNums(), messageFilter);
  if (getMessageResult != null) {
    response.setRemark(getMessageResult.getStatus().name());
    responseHeader.setNextBeginOffset(getMessageResult.getNextBeginOffset());
    responseHeader.setMinOffset(getMessageResult.getMinOffset());
    responseHeader.setMaxOffset(getMessageResult.getMaxOffset());

    if (getMessageResult.isSuggestPullingFromSlave()) {
      responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
    } else {
      responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
    }

    switch (this.brokerController.getMessageStoreConfig().getBrokerRole()) {
      case ASYNC_MASTER:
      case SYNC_MASTER:
        break;
      case SLAVE:
        if (!this.brokerController.getBrokerConfig().isSlaveReadEnable()) {
          response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
          responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
        }
        break;
    }

    if (this.brokerController.getBrokerConfig().isSlaveReadEnable()) {
      // consume too slow ,redirect to another machine
      if (getMessageResult.isSuggestPullingFromSlave()) {
        responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
      }
      // consume ok
      else {
        responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getBrokerId());
      }
    } else {
      responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
    }

    switch (getMessageResult.getStatus()) {
      case FOUND:
        response.setCode(ResponseCode.SUCCESS);
        break;
      case MESSAGE_WAS_REMOVING:
        response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
        break;
      case NO_MATCHED_LOGIC_QUEUE:
      case NO_MESSAGE_IN_QUEUE:
        if (0 != requestHeader.getQueueOffset()) {
          response.setCode(ResponseCode.PULL_OFFSET_MOVED);

          // XXX: warn and notify me
          log.info("the broker store no queue data, fix the request offset {} to {}, Topic: {} QueueId: {} Consumer Group: {}",
                   requestHeader.getQueueOffset(),
                   getMessageResult.getNextBeginOffset(),
                   requestHeader.getTopic(),
                   requestHeader.getQueueId(),
                   requestHeader.getConsumerGroup()
                  );
        } else {
          response.setCode(ResponseCode.PULL_NOT_FOUND);
        }
        break;
      case NO_MATCHED_MESSAGE:
        response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
        break;
      case OFFSET_FOUND_NULL:
        response.setCode(ResponseCode.PULL_NOT_FOUND);
        break;
      case OFFSET_OVERFLOW_BADLY:
        response.setCode(ResponseCode.PULL_OFFSET_MOVED);
        // XXX: warn and notify me
        log.info("the request offset: {} over flow badly, broker max offset: {}, consumer: {}",
                 requestHeader.getQueueOffset(), getMessageResult.getMaxOffset(), channel.remoteAddress());
        break;
      case OFFSET_OVERFLOW_ONE:
        response.setCode(ResponseCode.PULL_NOT_FOUND);
        break;
      case OFFSET_TOO_SMALL:
        response.setCode(ResponseCode.PULL_OFFSET_MOVED);
        log.info("the request offset too small. group={}, topic={}, requestOffset={}, brokerMinOffset={}, clientIp={}",
                 requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueOffset(),
                 getMessageResult.getMinOffset(), channel.remoteAddress());
        break;
      default:
        assert false;
        break;
    }

    if (this.hasConsumeMessageHook()) {
      ConsumeMessageContext context = new ConsumeMessageContext();
      context.setConsumerGroup(requestHeader.getConsumerGroup());
      context.setTopic(requestHeader.getTopic());
      context.setQueueId(requestHeader.getQueueId());

      String owner = request.getExtFields().get(BrokerStatsManager.COMMERCIAL_OWNER);

      switch (response.getCode()) {
        case ResponseCode.SUCCESS:
          int commercialBaseCount = brokerController.getBrokerConfig().getCommercialBaseCount();
          int incValue = getMessageResult.getMsgCount4Commercial() * commercialBaseCount;

          context.setCommercialRcvStats(BrokerStatsManager.StatsType.RCV_SUCCESS);
          context.setCommercialRcvTimes(incValue);
          context.setCommercialRcvSize(getMessageResult.getBufferTotalSize());
          context.setCommercialOwner(owner);

          break;
        case ResponseCode.PULL_NOT_FOUND:
          if (!brokerAllowSuspend) {

            context.setCommercialRcvStats(BrokerStatsManager.StatsType.RCV_EPOLLS);
            context.setCommercialRcvTimes(1);
            context.setCommercialOwner(owner);

          }
          break;
        case ResponseCode.PULL_RETRY_IMMEDIATELY:
        case ResponseCode.PULL_OFFSET_MOVED:
          context.setCommercialRcvStats(BrokerStatsManager.StatsType.RCV_EPOLLS);
          context.setCommercialRcvTimes(1);
          context.setCommercialOwner(owner);
          break;
        default:
          assert false;
          break;
      }

      this.executeConsumeMessageHookBefore(context);
    }

    switch (response.getCode()) {
      case ResponseCode.SUCCESS:

        this.brokerController.getBrokerStatsManager().incGroupGetNums(requestHeader.getConsumerGroup(), requestHeader.getTopic(),
                                                                      getMessageResult.getMessageCount());

        this.brokerController.getBrokerStatsManager().incGroupGetSize(requestHeader.getConsumerGroup(), requestHeader.getTopic(),
                                                                      getMessageResult.getBufferTotalSize());

        this.brokerController.getBrokerStatsManager().incBrokerGetNums(getMessageResult.getMessageCount());
        if (this.brokerController.getBrokerConfig().isTransferMsgByHeap()) {
          final long beginTimeMills = this.brokerController.getMessageStore().now();
          final byte[] r = this.readGetMessageResult(getMessageResult, requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueId());
          this.brokerController.getBrokerStatsManager().incGroupGetLatency(requestHeader.getConsumerGroup(),
                                                                           requestHeader.getTopic(), requestHeader.getQueueId(),
                                                                           (int) (this.brokerController.getMessageStore().now() - beginTimeMills));
          response.setBody(r);
        } else {
          try {
            FileRegion fileRegion =
              new ManyMessageTransfer(response.encodeHeader(getMessageResult.getBufferTotalSize()), getMessageResult);
            channel.writeAndFlush(fileRegion).addListener(new ChannelFutureListener() {
              @Override
              public void operationComplete(ChannelFuture future) throws Exception {
                getMessageResult.release();
                if (!future.isSuccess()) {
                  log.error("transfer many message by pagecache failed, {}", channel.remoteAddress(), future.cause());
                }
              }
            });
          } catch (Throwable e) {
            log.error("transfer many message by pagecache exception", e);
            getMessageResult.release();
          }

          response = null;
        }
        break;
      case ResponseCode.PULL_NOT_FOUND:

        if (brokerAllowSuspend && hasSuspendFlag) {
          long pollingTimeMills = suspendTimeoutMillisLong;
          if (!this.brokerController.getBrokerConfig().isLongPollingEnable()) {
            pollingTimeMills = this.brokerController.getBrokerConfig().getShortPollingTimeMills();
          }

          String topic = requestHeader.getTopic();
          long offset = requestHeader.getQueueOffset();
          int queueId = requestHeader.getQueueId();
          PullRequest pullRequest = new PullRequest(request, channel, pollingTimeMills,
                                                    this.brokerController.getMessageStore().now(), offset, subscriptionData, messageFilter);
          this.brokerController.getPullRequestHoldService().suspendPullRequest(topic, queueId, pullRequest);
          response = null;
          break;
        }

      case ResponseCode.PULL_RETRY_IMMEDIATELY:
        break;
      case ResponseCode.PULL_OFFSET_MOVED:
        if (this.brokerController.getMessageStoreConfig().getBrokerRole() != BrokerRole.SLAVE
            || this.brokerController.getMessageStoreConfig().isOffsetCheckInSlave()) {
          MessageQueue mq = new MessageQueue();
          mq.setTopic(requestHeader.getTopic());
          mq.setQueueId(requestHeader.getQueueId());
          mq.setBrokerName(this.brokerController.getBrokerConfig().getBrokerName());

          OffsetMovedEvent event = new OffsetMovedEvent();
          event.setConsumerGroup(requestHeader.getConsumerGroup());
          event.setMessageQueue(mq);
          event.setOffsetRequest(requestHeader.getQueueOffset());
          event.setOffsetNew(getMessageResult.getNextBeginOffset());
          this.generateOffsetMovedEvent(event);
          log.warn(
            "PULL_OFFSET_MOVED:correction offset. topic={}, groupId={}, requestOffset={}, newOffset={}, suggestBrokerId={}",
            requestHeader.getTopic(), requestHeader.getConsumerGroup(), event.getOffsetRequest(), event.getOffsetNew(),
            responseHeader.getSuggestWhichBrokerId());
        } else {
          responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getBrokerId());
          response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
          log.warn("PULL_OFFSET_MOVED:none correction. topic={}, groupId={}, requestOffset={}, suggestBrokerId={}",
                   requestHeader.getTopic(), requestHeader.getConsumerGroup(), requestHeader.getQueueOffset(),
                   responseHeader.getSuggestWhichBrokerId());
        }

        break;
      default:
        assert false;
    }
  } else {
    response.setCode(ResponseCode.SYSTEM_ERROR);
    response.setRemark("store getMessage return null");
  }

  boolean storeOffsetEnable = brokerAllowSuspend;
  storeOffsetEnable = storeOffsetEnable && hasCommitOffsetFlag;
  storeOffsetEnable = storeOffsetEnable
    && this.brokerController.getMessageStoreConfig().getBrokerRole() != BrokerRole.SLAVE;
  if (storeOffsetEnable) {
    this.brokerController.getConsumerOffsetManager().commitOffset(RemotingHelper.parseChannelRemoteAddr(channel),
                                                                  requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueId(), requestHeader.getCommitOffset());
  }
  return response;
}
```

## 从DefaultMessageStore中获取消息

DefaultMessageStore是broker对于消息存储的核心抽象。方法如下：

```java
public GetMessageResult getMessage(final String group, final String topic, final int queueId, final long offset,
                                   final int maxMsgNums,
                                   final MessageFilter messageFilter) {

  long beginTime = this.getSystemClock().now();
  GetMessageStatus status = GetMessageStatus.NO_MESSAGE_IN_QUEUE;
  long nextBeginOffset = offset;
  long minOffset = 0;
  long maxOffset = 0;

  GetMessageResult getResult = new GetMessageResult();

  final long maxOffsetPy = this.commitLog.getMaxOffset();
	//找到对应的ConsumeQueue和对应的最小最大offset
  ConsumeQueue consumeQueue = findConsumeQueue(topic, queueId);
  minOffset = consumeQueue.getMinOffsetInQueue();
  maxOffset = consumeQueue.getMaxOffsetInQueue();

  if (maxOffset == 0) {
    status = GetMessageStatus.NO_MESSAGE_IN_QUEUE;
    nextBeginOffset = nextOffsetCorrection(offset, 0);
  } else if (offset < minOffset) {
    status = GetMessageStatus.OFFSET_TOO_SMALL;
    nextBeginOffset = nextOffsetCorrection(offset, minOffset);
  } else if (offset == maxOffset) {
    status = GetMessageStatus.OFFSET_OVERFLOW_ONE;
    nextBeginOffset = nextOffsetCorrection(offset, offset);
  } else if (offset > maxOffset) {
    status = GetMessageStatus.OFFSET_OVERFLOW_BADLY;
    if (0 == minOffset) {
      nextBeginOffset = nextOffsetCorrection(offset, minOffset);
    } else {
      nextBeginOffset = nextOffsetCorrection(offset, maxOffset);
    }
  } else {
    //offset介于minOffset和maxOffset之间
    SelectMappedBufferResult bufferConsumeQueue = consumeQueue.getIndexBuffer(offset);
    if (bufferConsumeQueue != null) {
      try {
        status = GetMessageStatus.NO_MATCHED_MESSAGE;

        long nextPhyFileStartOffset = Long.MIN_VALUE;
        long maxPhyOffsetPulling = 0;

        int i = 0;
        final int maxFilterMessageCount = Math.max(16000, maxMsgNums * ConsumeQueue.CQ_STORE_UNIT_SIZE);
        final boolean diskFallRecorded = this.messageStoreConfig.isDiskFallRecorded();
        ConsumeQueueExt.CqExtUnit cqExtUnit = new ConsumeQueueExt.CqExtUnit();
        for (; i < bufferConsumeQueue.getSize() && i < maxFilterMessageCount; i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
          long offsetPy = bufferConsumeQueue.getByteBuffer().getLong();
          int sizePy = bufferConsumeQueue.getByteBuffer().getInt();
          long tagsCode = bufferConsumeQueue.getByteBuffer().getLong();

          maxPhyOffsetPulling = offsetPy;

          if (nextPhyFileStartOffset != Long.MIN_VALUE) {
            if (offsetPy < nextPhyFileStartOffset)
              continue;
          }

          boolean isInDisk = checkInDiskByCommitOffset(offsetPy, maxOffsetPy);

          if (this.isTheBatchFull(sizePy, maxMsgNums, getResult.getBufferTotalSize(), getResult.getMessageCount(),
                                  isInDisk)) {
            break;
          }

          boolean extRet = false, isTagsCodeLegal = true;
          if (consumeQueue.isExtAddr(tagsCode)) {
            extRet = consumeQueue.getExt(tagsCode, cqExtUnit);
            if (extRet) {
              tagsCode = cqExtUnit.getTagsCode();
            } else {
              // can't find ext content.Client will filter messages by tag also.
              log.error("[BUG] can't find consume queue extend file content!addr={}, offsetPy={}, sizePy={}, topic={}, group={}",
                        tagsCode, offsetPy, sizePy, topic, group);
              isTagsCodeLegal = false;
            }
          }

          if (messageFilter != null
              && !messageFilter.isMatchedByConsumeQueue(isTagsCodeLegal ? tagsCode : null, extRet ? cqExtUnit : null)) {
            if (getResult.getBufferTotalSize() == 0) {
              status = GetMessageStatus.NO_MATCHED_MESSAGE;
            }

            continue;
          }

          SelectMappedBufferResult selectResult = this.commitLog.getMessage(offsetPy, sizePy);
          if (null == selectResult) {
            if (getResult.getBufferTotalSize() == 0) {
              status = GetMessageStatus.MESSAGE_WAS_REMOVING;
            }

            nextPhyFileStartOffset = this.commitLog.rollNextFile(offsetPy);
            continue;
          }

          if (messageFilter != null
              && !messageFilter.isMatchedByCommitLog(selectResult.getByteBuffer().slice(), null)) {
            if (getResult.getBufferTotalSize() == 0) {
              status = GetMessageStatus.NO_MATCHED_MESSAGE;
            }
            // release...
            selectResult.release();
            continue;
          }

          this.storeStatsService.getGetMessageTransferedMsgCount().incrementAndGet();
          getResult.addMessage(selectResult);
          status = GetMessageStatus.FOUND;
          nextPhyFileStartOffset = Long.MIN_VALUE;
        }

        if (diskFallRecorded) {
          long fallBehind = maxOffsetPy - maxPhyOffsetPulling;
          brokerStatsManager.recordDiskFallBehindSize(group, topic, queueId, fallBehind);
        }

        nextBeginOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);

        long diff = maxOffsetPy - maxPhyOffsetPulling;
        long memory = (long) (StoreUtil.TOTAL_PHYSICAL_MEMORY_SIZE
                              * (this.messageStoreConfig.getAccessMessageInMemoryMaxRatio() / 100.0));
        getResult.setSuggestPullingFromSlave(diff > memory);
      } finally {

        bufferConsumeQueue.release();
      }
    } else {
      status = GetMessageStatus.OFFSET_FOUND_NULL;
      nextBeginOffset = nextOffsetCorrection(offset, consumeQueue.rollNextFile(offset));
      log.warn("consumer request topic: " + topic + "offset: " + offset + " minOffset: " + minOffset + " maxOffset: "
               + maxOffset + ", but access logic queue failed.");
    }
  }
  //统计命中或者miss的次数
  if (GetMessageStatus.FOUND == status) {
    this.storeStatsService.getGetMessageTimesTotalFound().incrementAndGet();
  } else {
    this.storeStatsService.getGetMessageTimesTotalMiss().incrementAndGet();
  }
  long elapsedTime = this.getSystemClock().now() - beginTime;
  this.storeStatsService.setGetMessageEntireTimeMax(elapsedTime);

  getResult.setStatus(status);
  getResult.setNextBeginOffset(nextBeginOffset);
  getResult.setMaxOffset(maxOffset);
  getResult.setMinOffset(minOffset);
  return getResult;
}
```



# 结语



(参考丁威、周继峰<<RocketMQ技术内幕>>。水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

