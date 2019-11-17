## Netty-Notes

### [Netty 服务端启动过程](https://github.com/martin-1992/Netty-Notes/tree/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B)
　　绑定端口最终会调用 dobind() 方法。

- [initAndRegister](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/initAndRegister.md)，创建 Channel、初始化配置 Channel、将 Channel 注册到 EventLoop（事件轮询器 Selector）；
- [doBind0](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/doBind0.md)，调用 JDK 底层 API 将端口与 [initAndRegister](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/initAndRegister.md) 创建好的 Channel 进行绑定，并添加监听器。

### [NioEventLoop](https://github.com/martin-1992/Netty-Notes/tree/master/NioEventLoop)

- [NioEventLoop 的创建](https://github.com/martin-1992/Netty-Notes/tree/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%88%9B%E5%BB%BA)；
- [NioEventLoop 的启动](https://github.com/martin-1992/Netty-Notes/tree/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%90%AF%E5%8A%A8)。

### [新连接的接入](https://github.com/martin-1992/Netty-Notes/tree/master/%E6%96%B0%E8%BF%9E%E6%8E%A5%E7%9A%84%E6%8E%A5%E5%85%A5)

- 在 NioEventLoop 的启动中，当 Channel 已经注册到 Selector 后，会调用 [NioEventLoop#processSelectedKey()](https://github.com/martin-1992/Netty-Notes/blob/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%90%AF%E5%8A%A8/processSelectedKeys.md) 对不同 IO 事件进行轮询处理。调用 [unsafe.read()](https://github.com/martin-1992/Netty-Notes/blob/master/%E6%96%B0%E8%BF%9E%E6%8E%A5%E7%9A%84%E6%8E%A5%E5%85%A5/%E6%A3%80%E6%B5%8B%E6%96%B0%E8%BF%9E%E6%8E%A5.md) 检测到 accpet 或 read 事件进行处理，主要是服务端 Channel 获取客户端的连接 Channel，将其包装成 [NioSocketChannel](https://github.com/martin-1992/Netty-Notes/blob/master/%E6%96%B0%E8%BF%9E%E6%8E%A5%E7%9A%84%E6%8E%A5%E5%85%A5/NioSocketChannel.md)，设置客户端的连接 Channel 设置禁止 Nagle 算法；
- 调用 pipeline#fireChannelRead 方法，会从头节点 head 开始往下传播，传播到 [ServerBootstrap#channelRead](https://github.com/martin-1992/Netty-Notes/blob/master/%E6%96%B0%E8%BF%9E%E6%8E%A5%E7%9A%84%E6%8E%A5%E5%85%A5/ServerBootstrap%23channelRead.md)，会将该客户端 Channel 注册到服务端 Channel 对应的 Selector 上；
- 当客户端连接 Channel 注册到 Selector 后，具体注册可看 [register](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/register.md)（这里客户端 Channel 注册到服务端 Selector 流程和服务端 Channel 注册到 Selector 的流程是类似的），会对该 Channel 调用 [pipeline#fireChannelActive](https://github.com/martin-1992/Netty-Notes/blob/master/%E6%96%B0%E8%BF%9E%E6%8E%A5%E7%9A%84%E6%8E%A5%E5%85%A5/pipeline%23fireChannelActive.md) 进行读事件注册，该方法包含两个方法。
  1. fireChannelActive，继续调用下个节点的 ChannelActive 方法；
  2. readIfIsAutoRead，从尾节点开始，往前调用 read 方法，最终调用头节点的 [DefaultChannelPipeline#read](https://github.com/martin-1992/Netty-Notes/blob/master/%E6%96%B0%E8%BF%9E%E6%8E%A5%E7%9A%84%E6%8E%A5%E5%85%A5/DefaultChannelPipeline%23read.md) 方法，将该 Channel 注册向 Selector 注册对读事件感兴趣，开始读取数据。

### [pipeline 解析](https://github.com/martin-1992/Netty-Notes/tree/master/pipeline%20%E8%A7%A3%E6%9E%90)
　　在[实例化 NioServerSocketChannel](https://github.com/martin-1992/Netty-Notes/blob/2571fcbffe2cb9588dddf59e76c9b885a0bf8458/%E6%96%B0%E8%BF%9E%E6%8E%A5%E7%9A%84%E6%8E%A5%E5%85%A5/NioServerSocketChannel.md) 时会创建 pipeline，使用哨兵模式，默认添加两个节点 [HeadContext](https://github.com/martin-1992/Netty-Notes/blob/master/pipeline%20%E8%A7%A3%E6%9E%90/HeadContext.md) 和 [TailContext](https://github.com/martin-1992/Netty-Notes/blob/master/pipeline%20%E8%A7%A3%E6%9E%90/TailContext.md)。

- [添加 ChannelHandler](https://github.com/martin-1992/Netty-Notes/blob/master/pipeline%20%E8%A7%A3%E6%9E%90/%E6%B7%BB%E5%8A%A0%20ChannelHandler/README.md)；

#### [Netty 解码](https://github.com/martin-1992/Netty-Notes/tree/master/Netty%20%E8%A7%A3%E7%A0%81)
　　Netty 解码是将一串二进制流解码成多个 ByteBuf（自定义协议数据包），然后交给业务逻辑进行处理。

#### [Recycler](https://github.com/martin-1992/Netty-Notes/tree/master/Recycler)
　　Recycler 为 Netty 的轻量级对象池，用于获取对象和用完回收对象。
