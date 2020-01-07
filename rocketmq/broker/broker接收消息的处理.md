# 概念

# 组件

# 用法



# 实现

```java
private RemotingCommand sendMessage(ChannelHandlerContext ctx,RemotingCommand request,
                                    SendMessageContext sendMessageContext,SendMessageRequestHeader requestHeader) {

  final RemotingCommand response = RemotingCommand.createResponseCommand(SendMessageResponseHeader.class);
  final SendMessageResponseHeader responseHeader = (SendMessageResponseHeader)response.readCustomHeader();

  response.setOpaque(request.getOpaque());

  response.addExtField(MessageConst.PROPERTY_MSG_REGION, this.brokerController.getBrokerConfig().getRegionId());
  response.addExtField(MessageConst.PROPERTY_TRACE_SWITCH, String.valueOf(this.brokerController.getBrokerConfig().isTraceOn()));

  final long startTimstamp = this.brokerController.getBrokerConfig().getStartAcceptSendRequestTimeStamp();
  if (this.brokerController.getMessageStore().now() < startTimstamp) {
    response.setCode(ResponseCode.SYSTEM_ERROR);
    response.setRemark(String.format("broker unable to service, until %s", UtilAll.timeMillisToHumanString2(startTimstamp)));
    return response;
  }

  response.setCode(-1);
  super.msgCheck(ctx, requestHeader, response);
  if (response.getCode() != -1) {
    return response;
  }

  final byte[] body = request.getBody();

  int queueIdInt = requestHeader.getQueueId();
  TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());

  if (queueIdInt < 0) {
    queueIdInt = Math.abs(this.random.nextInt() % 99999999) % topicConfig.getWriteQueueNums();
  }

  MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
  msgInner.setTopic(requestHeader.getTopic());
  msgInner.setQueueId(queueIdInt);

  if (!handleRetryAndDLQ(requestHeader, response, request, msgInner, topicConfig)) {
    return response;
  }

  msgInner.setBody(body);
  msgInner.setFlag(requestHeader.getFlag());
  MessageAccessor.setProperties(msgInner, MessageDecoder.string2messageProperties(requestHeader.getProperties()));
  msgInner.setBornTimestamp(requestHeader.getBornTimestamp());
  msgInner.setBornHost(ctx.channel().remoteAddress());
  msgInner.setStoreHost(this.getStoreHost());
  msgInner.setReconsumeTimes(requestHeader.getReconsumeTimes() == null ? 0 : requestHeader.getReconsumeTimes());
  String clusterName = this.brokerController.getBrokerConfig().getBrokerClusterName();
  MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_CLUSTER, clusterName);
  msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgInner.getProperties()));
  PutMessageResult putMessageResult = null;
  Map<String, String> oriProps = MessageDecoder.string2messageProperties(requestHeader.getProperties());
  String traFlag = oriProps.get(MessageConst.PROPERTY_TRANSACTION_PREPARED);
  if (traFlag != null && Boolean.parseBoolean(traFlag)) {
    if (this.brokerController.getBrokerConfig().isRejectTransactionMessage()) {
      response.setCode(ResponseCode.NO_PERMISSION);
      response.setRemark(
        "the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1()
        + "] sending transaction message is forbidden");
      return response;
    }
    putMessageResult = this.brokerController.getTransactionalMessageService().prepareMessage(msgInner);
  } else {
    putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);
  }

  return handlePutMessageResult(putMessageResult, response, request, msgInner, responseHeader, sendMessageContext, ctx, queueIdInt);

}
```



# 结语



(参考丁威、周继峰<<RocketMQ技术内幕>>。水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

