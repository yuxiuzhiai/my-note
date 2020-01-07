# 概念

# 组件

## AcceptSocketService(HAService的内部类)

用于在broker的master端监听slave的连接

## GroupTransferService(HAService的内部类)



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
//删除了try catch和日志代码
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
        } else {
          log.warn("Unexpected ops in select " + k.readyOps());
        }
      }
      selected.clear();
    }
  }
}
```

也可以看出，这就是java nio处理连接的方式

# 结语



(水平有限，最近在看rocketmq源码，记录学习过程，也希望对各位有点微小的帮助。如有错误，请指正~)

