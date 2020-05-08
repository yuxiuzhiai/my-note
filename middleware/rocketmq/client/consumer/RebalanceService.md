# 概念

在rocketmq中，消费者必须属于一个消费者组。消费者组有两种消费模式：

* 集群:就是一个消费者组内的消费者，每个消费一个topic的某个子集，所有的消费者的子集共同组成topic的所有消息
* 广播：就是消费者组内的所有消费者都消费topic的所有消息

我们还知道，在rocketmq中，订阅了某个topic，就相当于订阅了这个topic的所有消息队列，MessageQueue，而MessageQueue是可能动态变化的，消费者组内的消费者数量也可能动态变化。所以，就需要一个组件来完成这种消费队列在消费者间的重新平衡。这个组件就是RebalanceService。

# 实现

RebalanceService实现了Runnable接口，看看其重写的run方法：

```java
@Override
public void run() {
  while (!this.isStopped()) {
    //等待waitInterval时间
    this.waitForRunning(waitInterval);
    //主要的逻辑方法
    this.mqClientFactory.doRebalance();
  }
}
```

主要的逻辑方法在MQClientInstance.doRebalance()中实现

```java
public void doRebalance() {
  for (Map.Entry<String, MQConsumerInner> entry : this.consumerTable.entrySet()) {
    MQConsumerInner impl = entry.getValue();
    if (impl != null) {
      impl.doRebalance();
    }
  }
}
```

继续进入，找到RebalanceImpl.doRebalance()方法

```java
public void doRebalance(final boolean isOrder) {
  Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
  if (subTable != null) {
    for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) {
      final String topic = entry.getKey();
      //主要逻辑方法
      this.rebalanceByTopic(topic, isOrder);
    }
  }
  this.truncateMessageQueueNotMyTopic();
}
```

继续到rebalanceByTopic()方法

```java
private void rebalanceByTopic(final String topic, final boolean isOrder) {
  switch (messageModel) {
    case BROADCASTING: {
      Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
      if (mqSet != null) {
        boolean changed = this.updateProcessQueueTableInRebalance(topic, mqSet, isOrder);
        if (changed) {
          this.messageQueueChanged(topic, mqSet, mqSet);
        }
      }
      break;
    }
    case CLUSTERING: {
      //见下文 集群模式的处理
      break;
    }
    default:
      break;
  }
}
```

从代码来看，很清晰的根据消费模式的不同，集群模式和广播模式分别对应不同的处理

## 集群模式的处理

方法段为:

```java
//获取MessageQueue集合、消费者集合
Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup);

if (mqSet != null && cidAll != null) {
  List<MessageQueue> mqAll = new ArrayList<MessageQueue>();
  mqAll.addAll(mqSet);
  Collections.sort(mqAll);
  Collections.sort(cidAll);
	//根据MessageQueue分配策略获取MessageQueue
  AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;
  List<MessageQueue> allocateResult = strategy.allocate(this.consumerGroup,this.mQClientFactory.getClientId(),mqAll,cidAll);
  Set<MessageQueue> allocateResultSet = new HashSet<MessageQueue>();
  if (allocateResult != null) {
    allocateResultSet.addAll(allocateResult);
  }
	//检测MessageQueue相较于之前是否发生了变化
  boolean changed = this.updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder);
  if (changed) {
    this.messageQueueChanged(topic, mqSet, allocateResultSet);
  }
}
```

### 根据MessageQueue分配策略获取MessageQueue

AllocateMessageQueueStrategy定义了MessageQueue的分配策略。接口定义如下：

```java
public interface AllocateMessageQueueStrategy {
	//选择消息队列
  List<MessageQueue> allocate(final String consumerGroup,final String currentCID,final List<MessageQueue> mqAll,
                              final List<String> cidAll);
  //策略名称
  String getName();
}
```

在rocketmq中，有如下实现：

* AllocateMachineRoomNearby:根据机房位置
* AllocateMessageQueueAveragely：平均分
* AllocateMessageQueueAveragelyByCircle
* AllocateMessageQueueByConfig
* AllocateMessageQueueByMachineRoom
* AllocateMessageQueueConsistentHash：基于一致性hash算法

### 检测MessageQueue相较于之前是否发生了变化

大致逻辑为：

* 判断有没有删除老的MessageQueue，如果删除了，也从processQueueTable中删除
* 判断本次的MessageQueue，是不是有原来不存在的，如果有新增的MessageQueue，就构建一个PullRequest请求

```java
private boolean updateProcessQueueTableInRebalance(String topic,Set<MessageQueue> mqSet,boolean isOrder) {
  boolean changed = false;
  Iterator<Entry<MessageQueue, ProcessQueue>> it = this.processQueueTable.entrySet().iterator();
  //判断有没有删除老的MessageQueue
  while (it.hasNext()) {
    Entry<MessageQueue, ProcessQueue> next = it.next();
    MessageQueue mq = next.getKey();
    ProcessQueue pq = next.getValue();

    if (mq.getTopic().equals(topic)) {
      if (!mqSet.contains(mq)) {
        pq.setDropped(true);
        if (this.removeUnnecessaryMessageQueue(mq, pq)) {
          it.remove();
          changed = true;
        }
      } else if (pq.isPullExpired()) {
        switch (this.consumeType()) {
          case CONSUME_ACTIVELY:
            break;
          case CONSUME_PASSIVELY:
            pq.setDropped(true);
            if (this.removeUnnecessaryMessageQueue(mq, pq)) {
              it.remove();
              changed = true;
            }
            break;
          default:
            break;
        }
      }
    }
  }
  //判断有没有新增的MessageQueue
  List<PullRequest> pullRequestList = new ArrayList<PullRequest>();
  for (MessageQueue mq : mqSet) {
    if (!this.processQueueTable.containsKey(mq)) {
      if (isOrder && !this.lock(mq)) {
        continue;
      }
      this.removeDirtyOffset(mq);
      ProcessQueue pq = new ProcessQueue();
      //计算要从哪里开始消费
      long nextOffset = this.computePullFromWhere(mq);
      if (nextOffset >= 0) {
        ProcessQueue pre = this.processQueueTable.putIfAbsent(mq, pq);
        if (pre == null) {
          //构建PullRequest请求
          PullRequest pullRequest = new PullRequest();
          pullRequest.setXXX
            changed = true;
        }
      }
    }
    this.dispatchPullRequest(pullRequestList);
    return changed;
  }
}
```

#### 计算要从哪里开始消费

```java
public long computePullFromWhere(MessageQueue mq) {
  long result = -1;
  final ConsumeFromWhere consumeFromWhere = this.defaultMQPushConsumerImpl.getDefaultMQPushConsumer().getConsumeFromWhere();
  final OffsetStore offsetStore = this.defaultMQPushConsumerImpl.getOffsetStore();
  switch (consumeFromWhere) {
      //从MessageQueue的最新偏移量开始消费
    case CONSUME_FROM_LAST_OFFSET: 
      long lastOffset = offsetStore.readOffset(mq, ReadOffsetType.READ_FROM_STORE);
      if (lastOffset >= 0) {
        result = lastOffset;
      }
      // First start,no offset
      else if (-1 == lastOffset) {
        if (mq.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
          result = 0L;
        } else {
          result = this.mQClientFactory.getMQAdminImpl().maxOffset(mq);
        }
      } else {
        result = -1;
      }
      break;
    case CONSUME_FROM_FIRST_OFFSET: 
      //从MessageQueue的最小偏移量开始消费
      long lastOffset = offsetStore.readOffset(mq, ReadOffsetType.READ_FROM_STORE);
      if (lastOffset >= 0) {
        result = lastOffset;
      } else if (-1 == lastOffset) {
        result = 0L;
      } else {
        result = -1;
      }
      break;
    case CONSUME_FROM_TIMESTAMP: 
      //从消费者启动的时间戳对应的消费进度开始消费
      long lastOffset = offsetStore.readOffset(mq, ReadOffsetType.READ_FROM_STORE);
      if (lastOffset >= 0) {
        result = lastOffset;
      } else if (-1 == lastOffset) {
        if (mq.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
          result = this.mQClientFactory.getMQAdminImpl().maxOffset(mq);
        } else {
          long timestamp = UtilAll.parseDate(this.defaultMQPushConsumerImpl.getDefaultMQPushConsumer().getConsumeTimestamp(),
                                             UtilAll.YYYYMMDDHHMMSS).getTime();
          result = this.mQClientFactory.getMQAdminImpl().searchOffset(mq, timestamp);
        }
      } else {
        result = -1;
      }
      break;
  }
  return result;
}
```

ConsumeFromWhere是一个枚举，目前还有三个是没有Deprecated的值：

* CONSUME_FROM_LAST_OFFSET
* CONSUME_FROM_FIRST_OFFSET
* CONSUME_FROM_TIMESTAMP

不同的ConsumeFromWhere的第一步都是依赖OffsetStore先从本地读取offset，如果offset>-1，则表示是一个有效的值，则会已这个offset为准；如果offset=-1，是初始值，则需要根据不同的ConsumeFromWhere去broker读取对应MessageQueue的offset

# 结语

(水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

