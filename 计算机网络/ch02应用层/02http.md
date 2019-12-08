HTTP,超文本传输协议HyperText Transfer Protocol。基于TCP，无状态。

## 持续连接与非持续连接

http默认使用持续连接，但是也可以配置为使用非持续连接。持续连接就是指发送请求，接收响应后，并不直接关闭TCP连接，再有下个http请求，还用这个TCP连接发送；非持续连接就是发送请求，接收响应后，直接关闭TCP连接，后续的请求自己重新创建新的TCP连接

介绍一个概念，往返时间 Round-Trip Time RTT，是指一个短分组从客户到服务器然后再返回客户端所花费的时间。RTT包括分组传播时延、分组在中间路由器和交换机上的排队时延以及分组处理时延。

对于非持续连接，TCP三次握手的前两次消耗一个RTT，第三次握手时，客户端会向该TCP连接发送一个HTTP请求报文，然后服务器在该TCP连接上发送HTML文件。因此，一次请求会消耗两个RTT加上服务器传输HTML文件的时间

对于持续连接，在第一次三次握手建立TCP连接后，后续的HTTP请求就不需要消耗额外的RTT

## HTTP报文格式

请求报文



- 

  HTTP报文格式

- cookie

- web缓存

  CDN(Content Distribution Network)

- 条件GET

  普通GET请求 + If-Modified-Since头部即表示一个条件GET方法