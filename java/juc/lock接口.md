# 介绍

在Java中，最简单的锁的语义，就是synchronized关键字。synchronized关键字自己会处理锁的申请与释放，使用方便，但是在Java1.6之前，synchronized的性能并不好，是一个比较重量级的操作。但是随着Java的发展，Java也加入了多种锁机制，比如：偏向锁，轻量级锁，使得synchronized也并不那么的影响性能了。另一方面，synchronized关键字虽然使用方便，但是可操作性行并不强，对于一些更复杂的场景，并不能支持，比如：超时获取锁；尝试获取锁(获取失败就立马返回，线程不阻塞)。

so，Java中还有一个更具备操作性的锁的语义的实现者：Lock接口；以及一个支持共享锁的读写锁：ReadWriteLock接口

# 相关接口和实现

## 关联接口

### Lock接口

```java
public interface Lock {
  //获取锁
  void lock();
  //可中断获取锁
  void lockInterruptibly() throws InterruptedException;
  //尝试获取锁。即如果当前锁没有被占用就获取成功，否则就获取失败，但是无论如何方法会立即返回，线程不会阻塞住
  boolean tryLock();
  //超时尝试获取锁
  boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
  //释放锁
  void unlock();
  //返回当前锁实例的一个Condition对象实例。
  Condition newCondition();
}
```

既然接口中涉及到了Condition接口，那就顺带也说一下。

### Condition接口

```java
public interface Condition {
  //使当前线程在某个条件上等待，除非被唤醒或者中断，否则会一直等待。调用这个方法的前提是当前线程必须持有锁。调用这个方法会使当前线程释放锁
  void await() throws InterruptedException;
	//同上，但是不响应中断
  void awaitUninterruptibly();
	//有时限的等待
  long awaitNanos(long nanosTimeout) throws InterruptedException;
  boolean await(long time, TimeUnit unit) throws InterruptedException;
  boolean awaitUntil(Date deadline) throws InterruptedException;
	//唤醒一个等待的线程
  void signal();
	//唤醒所有等待的线程
  void signalAll();
}
```

### ReadWriteLock

读写锁，或者说是排它锁与共享锁。

```java
public interface ReadWriteLock {
  //获取读锁，或者说是共享锁
  Lock readLock();
	//获取写锁，或者说是排它锁
  Lock writeLock();
}
```

## juc中的实现类

### ReentrantLock

可重入锁，应该是Java中除了synchronized关键字，最常使用的锁了。内部基于AQS实现，具备公平锁和不公平锁两种机制

### ReentrantReadWriteLock

ReadWriteLock的实现类。

### StampedLock

1.8新增，并没有实现Lock接口。

# 实现

## ReentrantLock的实现

