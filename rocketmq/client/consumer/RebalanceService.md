# 概念

# 组件

## AllocateMessageQueueStrategy

分配MessageQueue的策略。接口定义如下：

```java
public interface AllocateMessageQueueStrategy {

  /**
     * Allocating by consumer id
     *
     * @param consumerGroup 当前的消费者组
     * @param currentCID 当前消费者id
     * @param mqAll 当前topic的所有MessageQueue
     * @param cidAll 当前消费者组的所有消费者id
     * @return 当前消费者分配到的MessageQueue
     */
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

# 用法



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
      try {
        impl.doRebalance();
      } catch (Throwable e) {
        log.error("doRebalance exception", e);
      }
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
      try {
        //主要逻辑方法
        this.rebalanceByTopic(topic, isOrder);
      } catch (Throwable e) {
        if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
          log.warn("rebalanceByTopic Exception", e);
        }
      }
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
          log.info("messageQueueChanged {} {} {} {}",
                   consumerGroup,
                   topic,
                   mqSet,
                   mqSet);
        }
      } else {
        log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
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
//1.获取MessageQueue集合、消费者集合
Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup);

if (mqSet != null && cidAll != null) {
  List<MessageQueue> mqAll = new ArrayList<MessageQueue>();
  mqAll.addAll(mqSet);

  Collections.sort(mqAll);
  Collections.sort(cidAll);
	//2.根据MessageQueue分配策略获取MessageQueue
  AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;
  List<MessageQueue> allocateResult = strategy.allocate(this.consumerGroup,this.mQClientFactory.getClientId(),mqAll,cidAll);
  Set<MessageQueue> allocateResultSet = new HashSet<MessageQueue>();
  if (allocateResult != null) {
    allocateResultSet.addAll(allocateResult);
  }
	//3.检测MessageQueue相较于之前是否发生了变化
  boolean changed = this.updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder);
  if (changed) {
    //删除了输出日志的代码
    //
    this.messageQueueChanged(topic, mqSet, allocateResultSet);
  }
}
```

### 1.获取MessageQueue集合、消费者集合

### 2.根据MessageQueue分配策略获取MessageQueue

### 3.检测MessageQueue相较于之前是否发生了变化

大致逻辑为：

* 判断有没有删除老的MessageQueue，如果删除了，也从processQueueTable中删除
* 判断本次的MessageQueue，是不是有原来不存在的，如果有新增的MessageQueue，就构建一个PullRequest请求

```java
private boolean updateProcessQueueTableInRebalance(final String topic, final Set<MessageQueue> mqSet,
                                                   final boolean isOrder) {
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
          log.info("doRebalance, {}, remove unnecessary mq, {}", consumerGroup, mq);
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
        log.warn("doRebalance, {}, add a new mq failed, {}, because lock failed", consumerGroup, mq);
        continue;
      }

      this.removeDirtyOffset(mq);
      ProcessQueue pq = new ProcessQueue();
      long nextOffset = this.computePullFromWhere(mq);
      if (nextOffset >= 0) {
        ProcessQueue pre = this.processQueueTable.putIfAbsent(mq, pq);
        if (pre != null) {
          log.info("doRebalance, {}, mq already exists, {}", consumerGroup, mq);
        } else {
          //构建PullRequest请求
          log.info("doRebalance, {}, add a new mq, {}", consumerGroup, mq);
          PullRequest pullRequest = new PullRequest();
          pullRequest.setConsumerGroup(consumerGroup);
          pullRequest.setNextOffset(nextOffset);
          pullRequest.setMessageQueue(mq);
          pullRequest.setProcessQueue(pq);
          pullRequestList.add(pullRequest);
          changed = true;
        }
      } else {
        log.warn("doRebalance, {}, add new mq failed, {}", consumerGroup, mq);
      }
    }
  }

  this.dispatchPullRequest(pullRequestList);

  return changed;
}
```





# 结语



(水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

