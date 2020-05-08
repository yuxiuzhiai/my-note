# 概念

我们知道，生产者发送消息，需要发送到指定的MessageQueue上，如果发送失败了，则很可能说明这个MessageQueue所在的broker出现了某种问题，则在发送下一条消息或者重试的时候，需要尽可能的避免上次失败的broker。在rocketmq中，MQFaultStrategy负责做这件事情。

# 组件

## LatencyFaultTolerance

用于判断一个broker是否有啥毛病。有一个默认实现LatencyFaultToleranceImpl。内部有一个FaultItem的Map，如果哪一次消息发送失败，如果开启了故障延迟机制，那就会构造一条FaultItem记录，已brokerName为key加入到这个map中去，那在某个时刻之前，这个brokerName都会当做是故障的

### FaultItem

LatencyFaultToleranceImpl的内部类。主要有三个属性：

* name：brokerName
* currentLatency：这次发送消息到出现异常的时间
* startTimestamp：在这个时间点以前，这个brokerName都会标记为故障

# 用途

MQFaultStrategy有两个作用：

* 用于挑选MessageQueue
* 如果开启了故障延迟机制，则在消息发送失败的时候，会生成一个FaultItem，标记broker故障

# 实现

挑选MessageQueue的实现在讲解producer的时候已经说了。这里仅仅看MQFaultStrategy.updateFaultItem()方法

```java
public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
  //如果开启了故障延迟机制
  if (this.sendLatencyFaultEnable) {
    //1.计算故障延迟的时间
    long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);
    //2.更新故障记录map
    this.latencyFaultTolerance.updateFaultItem(brokerName, currentLatency, duration);
  }
}
```

## 1.计算故障延迟的时间

```java
private long computeNotAvailableDuration(final long currentLatency) {
  for (int i = latencyMax.length - 1; i >= 0; i--) {
    if (currentLatency >= latencyMax[i])
      return this.notAvailableDuration[i];
  }
  return 0;
}
//两个数组：
//private long[] latencyMax = {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L};
//private long[] notAvailableDuration = {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L};
```

根据本次发送消息的时间，选择一个对应的故障延迟时间

## 2.更新故障记录map

```java
public void updateFaultItem(final String name, final long currentLatency, final long notAvailableDuration) {
  //看看有没有这个brokerName的故障记录
  FaultItem old = this.faultItemTable.get(name);
  if (null == old) {
    //没有brokerName已存的故障记录，新建一个，存入map
    final FaultItem faultItem = new FaultItem(name);
    faultItem.setCurrentLatency(currentLatency);
    faultItem.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);

    old = this.faultItemTable.putIfAbsent(name, faultItem);
    if (old != null) {
      old.setCurrentLatency(currentLatency);
      old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
    }
  } else {
    //已有故障记录，更新
    old.setCurrentLatency(currentLatency);
    old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
  }
}
```

# 结语



(参考丁威、周继峰<<RocketMQ技术内幕>>。水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

