# 概念

在rocketmq中，具体消费消息这一行为，由ConsumeMessageService实现。

## 类结构

接口定义：

```java
public interface ConsumeMessageService {
  void start();

  void shutdown();

  void updateCorePoolSize(int corePoolSize);

  void incCorePoolSize();

  void decCorePoolSize();

  int getCorePoolSize();
	//直接消费消息
  ConsumeMessageDirectlyResult consumeMessageDirectly(final MessageExt msg, final String brokerName);
	//提交消费消息的请求
  void submitConsumeRequest(
    final List<MessageExt> msgs,
    final ProcessQueue processQueue,
    final MessageQueue messageQueue,
    final boolean dispathToConsume);
}
```

从接口定义不难看出，其具体实现肯定跟线程池有关。

在rocketmq里面的具体实现

* ConsumeMessageConcurrentlyService：并发消费消息
* ConsumeMessageOrderlyService：顺序消费消息

# 组件

## MessageListener

用于接收异步传递的消息，是一个标记接口。根据并发消费和顺序消费两种方式，有两个子接口：

* MessageListenerConcurrently

  ```java
  public interface MessageListenerConcurrently extends MessageListener {
      ConsumeConcurrentlyStatus consumeMessage(final List<MessageExt> msgs,final ConsumeConcurrentlyContext context);
  }
  ```

  

* MessageListenerOrderly

  ```java
  public interface MessageListenerOrderly extends MessageListener {
      ConsumeOrderlyStatus consumeMessage(final List<MessageExt> msgs,final ConsumeOrderlyContext context);
  }
  ```

具体的消费逻辑，就是通过实现MessageListener接口来实现的

# 实现

从ConsumeMessageService的实现类可以看出，在rocketmq中，消息的消费有两种：一种是并发消费，另一种是顺序消费。我们挨个来分析分析

## 并发消费

直接看ConsumeMessageConcurrentlyService.submitConsumeRequest()方法：

```java
public void submitConsumeRequest(List<MessageExt> msgs,ProcessQueue processQueue,MessageQueue messageQueue,
                                 boolean dispatchToConsume) {
  final int consumeBatchSize = this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();
  //如果消息条数<=consumeBatchSize,则一个ConsumeRequest任务就可以了，提交到线程池
  if (msgs.size() <= consumeBatchSize) {
    ConsumeRequest consumeRequest = new ConsumeRequest(msgs, processQueue, messageQueue);
    this.consumeExecutor.submit(consumeRequest);
  } else {
    //如果消息条数>consumeBatchSize,则分页，分成多个ConsumeRequest任务，提交到线程池
    for (int total = 0; total < msgs.size(); ) {
      List<MessageExt> msgThis = new ArrayList<MessageExt>(consumeBatchSize);
      for (int i = 0; i < consumeBatchSize; i++, total++) {
        if (total < msgs.size()) {
          msgThis.add(msgs.get(total));
        } else {
          break;
        }
      }
      ConsumeRequest consumeRequest = new ConsumeRequest(msgThis, processQueue, messageQueue);
      this.consumeExecutor.submit(consumeRequest);
    }
  }
}
```

这里的逻辑，根据本次获取到的消息数量，看看要不要分页，归根结底就是构造出ConsumeRequest任务，提交到线程池中去。既然COnsumeRequest是一个任务，我们就再来看看他的run()方法都干了什么。

```java
public void run() {
  MessageListenerConcurrently listener = ConsumeMessageConcurrentlyService.this.messageListener;
  ConsumeConcurrentlyContext context = new ConsumeConcurrentlyContext(messageQueue);
  ConsumeConcurrentlyStatus status = null;
  //重试消息主题名？？？
  defaultMQPushConsumerImpl.resetRetryAndNamespace(msgs, defaultMQPushConsumer.getConsumerGroup());

  ConsumeMessageContext consumeMessageContext = null;
  if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
    consumeMessageContext = new ConsumeMessageContext();
    consumeMessageContext.setNamespace(defaultMQPushConsumer.getNamespace());
    consumeMessageContext.setConsumerGroup(defaultMQPushConsumer.getConsumerGroup());
    consumeMessageContext.setProps(new HashMap<String, String>());
    consumeMessageContext.setMq(messageQueue);
    consumeMessageContext.setMsgList(msgs);
    consumeMessageContext.setSuccess(false);
    ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext);
  }

  long beginTimestamp = System.currentTimeMillis();
  boolean hasException = false;
  ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;

  if (msgs != null && !msgs.isEmpty()) {
    for (MessageExt msg : msgs) {
      MessageAccessor.setConsumeStartTimeStamp(msg, String.valueOf(System.currentTimeMillis()));
    }
  }
  status = listener.consumeMessage(Collections.unmodifiableList(msgs), context);

  long consumeRT = System.currentTimeMillis() - beginTimestamp;
  if (null == status) {
    if (hasException) {
      returnType = ConsumeReturnType.EXCEPTION;
    } else {
      returnType = ConsumeReturnType.RETURNNULL;
    }
  } else if (consumeRT >= defaultMQPushConsumer.getConsumeTimeout() * 60 * 1000) {
    returnType = ConsumeReturnType.TIME_OUT;
  } else if (ConsumeConcurrentlyStatus.RECONSUME_LATER == status) {
    returnType = ConsumeReturnType.FAILED;
  } else if (ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status) {
    returnType = ConsumeReturnType.SUCCESS;
  }

  if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
    consumeMessageContext.getProps().put(MixAll.CONSUME_CONTEXT_TYPE, returnType.name());
  }

  if (null == status) {
    log.warn("consumeMessage return null, Group: {} Msgs: {} MQ: {}",
             ConsumeMessageConcurrentlyService.this.consumerGroup,
             msgs,
             messageQueue);
    status = ConsumeConcurrentlyStatus.RECONSUME_LATER;
  }

  if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
    consumeMessageContext.setStatus(status.toString());
    consumeMessageContext.setSuccess(ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status);
    ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext);
  }

  ConsumeMessageConcurrentlyService.this.getConsumerStatsManager()
    .incConsumeRT(ConsumeMessageConcurrentlyService.this.consumerGroup, messageQueue.getTopic(), consumeRT);

  if (!processQueue.isDropped()) {
    ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);
  } else {
    log.warn("processQueue is dropped without process consume result. messageQueue={}, msgs={}", messageQueue, msgs);
  }
}
```



## 顺序消费

```java

public void submitConsumeRequest(List<MessageExt> msgs,ProcessQueue processQueue,MessageQueue messageQueue, 
                                 boolean dispathToConsume) {
  if (dispathToConsume) {
    ConsumeRequest consumeRequest = new ConsumeRequest(processQueue, messageQueue);
    this.consumeExecutor.submit(consumeRequest);
  }
}
```



# 结语



(水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

