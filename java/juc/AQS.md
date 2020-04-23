# 介绍

在Java中，除了synchronized关键字具备锁的语义，还有juc包中的Lock接口。在观察Lock接口的多个不同实现后，不难发现，其内部锁的语义的实现，基本都是仰仗着AbstractQueuedSynchronizer，简称AQS。

# 内部状态

## Node

既然要实现锁的语义，则必须处理获取竞争锁失败，线程等待的情况。在AQS线程获取锁失败，就会构造一个Node接口，放入内部维护的同步队列中。

### 字段

* int waitStatus：等待状态。有以下几个值。
  * 1：当前线程被取消了
  * -1：标识当前节点的后继节点中的线程当前是被阻塞的，所以当前节点在出队时应该唤醒后继节点
  * -2：当前节点在某个Condition的等待队列中
  * -3：仅头节点可能设置为这个标识，代表doReleaseShared方法应该传播下去
  * 0：区别以上的状态值
* Node prev：前驱节点

* Node next：后继节点

* nextWaiter：
* Thread thread：当前节点里面的线程

## state

AQS内部有个int类型的变量state。需要注意：state是volatile变量。在AQS中，锁的获取和释放就表现在state值的变化。

# 实现

既然AQS的主要作用是用来辅助各种锁的实现类的。那它自研必然也会有相关的锁的申请和释放的方法。而锁又分为共享锁和排它锁，因此，AQS内部也有4个对应的方法：

排它锁：

* acquire(int arg):排它锁的申请
* release(int arg)：排它锁的释放

共享锁：

* acquireShared(int arg):共享锁的申请
* releaseShared(int arg):共享锁的释放

下面逐个分析每个方法。

## 排它锁的申请：acquire(int arg)方法

acquire(int arg)是一个模板方法，尝试获取锁的方法tryAcquire()是由子类负责实现。

```java
public final void acquire(int arg) {
  //1.
  if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

### 1.尝试获取锁：tryAcquire(int arg)方法

这是一个抽象方法。子类的实现一般都是以CAS的方式去尝试改变state的值。如果CAS设置值成功，则为成功获取到了锁，方法直接退出；如果没有获取到锁，则会执行后面的方法

### 2.如果获取锁失败，则创建一个节点：addWaiter(Node node)方法

对于获取排它锁而言，这里构造Node时，传入的waiter是Node.EXCLUSIVE（其实值为null，仅仅是标记是排它锁）

```java
private Node addWaiter(Node mode) {
  //创建一个Node，waiter是Node.EXCLUSIVE(其实就是null)
  Node node = new Node(Thread.currentThread(), mode);
  Node pred = tail;
  if (pred != null) {
    //把原来的尾节点设置为这次构造的Node的前驱节点
    node.prev = pred;
    //以cas的方式将尾节点设置为当前节点。如果设置成功，则返回，方法结束
    if (compareAndSetTail(pred, node)) {
      pred.next = node;
      return node;
    }
  }
  //如果前面cas设置尾结点失败，则用循环CAS的方式将Node加入到队列中
  enq(node);
  return node;
}
```

### 3.自旋，尝试获取锁：acquireQueued()方法

```java
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    boolean interrupted = false;
    for (;;) {
      //获取当前节点的前驱节点
      final Node p = node.predecessor();
      //如果前驱节点为头节点，则尝试获取锁
      if (p == head && tryAcquire(arg)) {
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return interrupted;
      }
      //如果获取锁失败
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

#### 3.1.获取锁失败后，判断是否要挂起当前node里的线程

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  int ws = pred.waitStatus;
  if (ws == Node.SIGNAL)
    //SIGNAL状态表示当前Node的后继节点需要被notify，所以如果是这个状态，则可以阻塞node
    return true;
  if (ws > 0) {
    //waitStatus>0的值只有一个 = 1，表示前驱节点以及被取消，则将当前节点的前驱节点设置为更前面的一个非取消节点
    do {
      node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;
  } else {
    /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}
```

#### 3.2. 如果需要阻塞当前线程，则阻塞，并返回线程是否被中断

## 排它锁的释放：release(int arg)方法

release()也是个模板方法，尝试释放锁的tryRelease()方法也需要子类自己实现

```java
public final boolean release(int arg) {
	//尝试释放锁
  if (tryRelease(arg)) {
    //释放成功
    Node h = head;
    if (h != null && h.waitStatus != 0)
      //唤醒后继节点
      unparkSuccessor(h);
    return true;
  }
  //释放失败
  return false;
}
```

## 共享锁的获取：acquireShared(int arg)方法

```java
public final void acquireShared(int arg) {
  //tryAcquireShared方法由子类负责重写
  if (tryAcquireShared(arg) < 0)
    //获取失败
    doAcquireShared(arg);
}
```

asd 

```java
private void doAcquireShared(int arg) {
  //构造节点，waiter是Node.SHARED,并以CAS的方式加入到同步队列尾部
  final Node node = addWaiter(Node.SHARED);
  boolean failed = true;
  try {
    boolean interrupted = false;
    for (;;) {
      //获取前驱节点
      final Node p = node.predecessor();
      //前驱节点是头结点
      if (p == head) {
        //尝试获取共享锁
        int r = tryAcquireShared(arg);
        //获取共享锁成功
        if (r >= 0) {
         	
          setHeadAndPropagate(node, r);
          p.next = null; // help GC
          if (interrupted)
            selfInterrupt();
          failed = false;
          return;
        }
      }
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

## 共享锁的释放：releaseShared(int arg)

```java
public final boolean releaseShared(int arg) {
  if (tryReleaseShared(arg)) {
    doReleaseShared();
    return true;
  }
  return false;
}
```

主要逻辑在doReleaseShared()方法：

```java
private void doReleaseShared() {
  for (;;) {
    Node h = head;
    if (h != null && h != tail) {
      int ws = h.waitStatus;
      if (ws == Node.SIGNAL) {
        if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
          continue;
        //唤醒后继节点
        unparkSuccessor(h);
      } else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))//如果waitStatus=0且cas设置h的状态失败在继续下一次循环
        continue;                
    }
    if (h == head)                   // loop if head changed
      break;
  }
}
```





