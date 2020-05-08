# 概念

# 组件

## AcceptSocketService(HAService的内部类)

用于在broker的master端监听slave的连接

## GroupTransferService(HAService的内部类)

## HAClient

是主从同步时，slave的核心实现类

# 实现

## 启动

先看看HAService的启动。

```java
public void start() throws Exception {
  //1.启动本地端口，开始监听
  this.acceptSocketService.beginAccept();
  //2.监听连接事件
  this.acceptSocketService.start();
  //3.
  this.groupTransferService.start();
  //4.
  this.haClient.start();
}
```

### 1.启动本地端口，开始监听

```java
public void beginAccept() throws Exception {
  this.serverSocketChannel = ServerSocketChannel.open();
  this.selector = RemotingUtil.openSelector();
  this.serverSocketChannel.socket().setReuseAddress(true);
  this.serverSocketChannel.socket().bind(this.socketAddressListen);
  this.serverSocketChannel.configureBlocking(false);
  this.serverSocketChannel.register(this.selector, SelectionKey.OP_ACCEPT);
}
```

看到这熟悉的ServerSocketChannel，Selector，没错，这就是以原生的java nio的方法监听一个端口

### 2.监听连接事件

AcceptSocketService同时也继承了ServiceThread，start()方法就是执行Runnable的run()方法，如下:

```java
public void run() {
  while (!this.isStopped()) {
    this.selector.select(1000);
    Set<SelectionKey> selected = this.selector.selectedKeys();
    if (selected != null) {
      for (SelectionKey k : selected) {
        if ((k.readyOps() & SelectionKey.OP_ACCEPT) != 0) {
          SocketChannel sc = ((ServerSocketChannel) k.channel()).accept();
          if (sc != null) {
            HAConnection conn = new HAConnection(HAService.this, sc);
            conn.start();
            HAService.this.addConnection(conn);
          }
        }
      }
      selected.clear();
    }
  }
}
```

也可以看出，这就是java nio处理连接的方式

### 3.

GroupTransferService也是继承了ServiceThread，它的start()方法也是执行run()方法，进入：

```java
public void run() {
  while (!this.isStopped()) {
      this.waitForRunning(10);
      this.doWaitTransfer();
  }
}
```

主要逻辑在this.doWaitTransfer()方法中，进入：

```java
private void doWaitTransfer() {
  synchronized (this.requestsRead) {
    if (!this.requestsRead.isEmpty()) {
      for (CommitLog.GroupCommitRequest req : this.requestsRead) {
        boolean transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
        long waitUntilWhen = HAService.this.defaultMessageStore.getSystemClock().now()
          + HAService.this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout();
        while (!transferOK && HAService.this.defaultMessageStore.getSystemClock().now() < waitUntilWhen) {
          this.notifyTransferObject.waitForRunning(1000);
          transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
        }
        req.wakeupCustomer(transferOK);
      }
      this.requestsRead.clear();
    }
  }
}
```

### 4.slave核心类启动

HAClient是slave的核心。看看其run()方法的实现：

```java
public void run() {
  while (!this.isStopped()) {
    //4.1尝试连接master
    if (this.connectMaster()) {
      //判断是否需要向master上报当前的最大偏移,已时间间隔来判断
      if (this.isTimeToReportOffset()) {
        //4.2.向master上报最大偏移
        boolean result = this.reportSlaveMaxOffset(this.currentReportedOffset);
      }
     	//selector等待socket事件
      this.selector.select(1000);
      //4.3.处理master的响应
      boolean ok = this.processReadEvent();
      if (!reportSlaveMaxOffsetPlus()) {
        continue;
      }
      long interval = HAService.this.getDefaultMessageStore().getSystemClock().now() - this.lastWriteTimestamp;
      if (interval > HAService.this.getDefaultMessageStore().getMessageStoreConfig().getHaHousekeepingInterval()) {
        this.closeMaster();
      }
    } else {
      this.waitForRunning(1000 * 5);
    }
  }
}
```

#### 4.1.连接master

```java
private boolean connectMaster() throws ClosedChannelException {
  if (null == socketChannel) {
    //获取master地址
    String addr = this.masterAddress.get();
    if (addr != null) {
      SocketAddress socketAddress = RemotingUtil.string2SocketAddress(addr);
      if (socketAddress != null) {
        //建立到master的tcp连接
        this.socketChannel = RemotingUtil.connect(socketAddress);
        if (this.socketChannel != null) {
          //注册读事件
          this.socketChannel.register(this.selector, SelectionKey.OP_READ);
        }
      }
    }
    this.currentReportedOffset = HAService.this.defaultMessageStore.getMaxPhyOffset();
    this.lastWriteTimestamp = System.currentTimeMillis();
  }
  return this.socketChannel != null;
}
```

#### 4.2.向master上报最大偏移

```java
private boolean reportSlaveMaxOffset(final long maxOffset) {
  this.reportOffset.position(0);
  this.reportOffset.limit(8);
  this.reportOffset.putLong(maxOffset);
  this.reportOffset.position(0);
  this.reportOffset.limit(8);

  for (int i = 0; i < 3 && this.reportOffset.hasRemaining(); i++) {
    this.socketChannel.write(this.reportOffset);
  }
  lastWriteTimestamp = HAService.this.defaultMessageStore.getSystemClock().now();
  return !this.reportOffset.hasRemaining();
}
```

#### 4.3.处理master的响应

```java
private boolean processReadEvent() {
  int readSizeZeroTimes = 0;
  while (this.byteBufferRead.hasRemaining()) {
    int readSize = this.socketChannel.read(this.byteBufferRead);
    if (readSize > 0) {
      readSizeZeroTimes = 0;
      //
      boolean result = this.dispatchReadRequest();
      if (!result) {
        return false;
      }
    } else if (readSize == 0) {
      if (++readSizeZeroTimes >= 3) {
        break;
      }
    } else {
      return false;
    }
  }
  return true;
}
```

##### 分发请求

```java
private boolean dispatchReadRequest() {
  final int msgHeaderSize = 8 + 4; // phyoffset + size
  int readSocketPos = this.byteBufferRead.position();

  while (true) {
    int diff = this.byteBufferRead.position() - this.dispatchPosition;
    if (diff >= msgHeaderSize) {
      long masterPhyOffset = this.byteBufferRead.getLong(this.dispatchPosition);
      int bodySize = this.byteBufferRead.getInt(this.dispatchPosition + 8);

      long slavePhyOffset = HAService.this.defaultMessageStore.getMaxPhyOffset();

      if (slavePhyOffset != 0) {
        if (slavePhyOffset != masterPhyOffset) {
          return false;
        }
      }

      if (diff >= (msgHeaderSize + bodySize)) {
        byte[] bodyData = new byte[bodySize];
        this.byteBufferRead.position(this.dispatchPosition + msgHeaderSize);
        this.byteBufferRead.get(bodyData);

        HAService.this.defaultMessageStore.appendToCommitLog(masterPhyOffset, bodyData);

        this.byteBufferRead.position(readSocketPos);
        this.dispatchPosition += msgHeaderSize + bodySize;

        if (!reportSlaveMaxOffsetPlus()) {
          return false;
        }

        continue;
      }
    }

    if (!this.byteBufferRead.hasRemaining()) {
      this.reallocateByteBuffer();
    }

    break;
  }

  return true;
}
```



# 结语



(水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

