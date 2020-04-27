## Netty-Notes

### [reactor 三种模式](https://github.com/martin-1992/Netty-Notes/tree/master/reactor%20%E4%B8%89%E7%A7%8D%E6%A8%A1%E5%BC%8F)

- BIO，为传统的 thread per connection，阻塞 I/O。类似排队打饭，一个连接数据处理完，在处理下一个连接；
- NIO，为 Reactor 模式，非阻塞 I/O。类似点菜、等待被叫，先接收连接，当有请求时在去处理；
- AIO，为 Procator 模式，异步 I/O。类似包厢，接收连接，当数据处理好通过回调给程序。

### [Netty 服务端启动过程](https://github.com/martin-1992/Netty-Notes/tree/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B)
　　绑定端口最终会调用 dobind() 方法。

- initAndRegister，创建 Channel、初始化配置 Channel、将 Channel 注册到 EventLoop（事件轮询器 Selector）；
- doBind0，调用 JDK 底层 API 将端口与 initAndRegister 创建好的 Channel 进行绑定，并添加监听器。

### [NioEventLoop 的创建](https://github.com/martin-1992/Netty-Notes/tree/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%88%9B%E5%BB%BA)

- 创建一个优化后的创建一个 Selector（IO 事件轮询器）；
- 创建线程工厂 Executor；
- 创建任务队列 taskQueue；
- 设置了队列的拒绝策略。

### [NioEventLoop 的启动](https://github.com/martin-1992/Netty-Notes/tree/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%90%AF%E5%8A%A8)

- 在 Netty 服务端启动过程讲到服务端 Channel 注册到 Selector 的某个 NioEventLoop 上，这时 **NioEventLoop 会调用 run() 方法进行死循环，处理 IO 事件。** IO 事件分为以下几种；
    1. 客户端的新连接的接入；
    2. 读取客户端连接的数据；
    3. 回复请求，将请求数据写回到客户端。
- 因为服务端 Channel 注册时，设置感兴趣的事件为 OP_ACCEPT，所以会接收客户端 Channel，同样重复服务端 Channel 的注册流程，不过客户端 Channel 会注册到 workerGroup 上的 NioEventLoop。

#### [客户端的新连接的接入](https://github.com/martin-1992/Netty-Notes/tree/master/%E6%96%B0%E8%BF%9E%E6%8E%A5%E7%9A%84%E6%8E%A5%E5%85%A5)
　　在 NioEventLoop 的启动中，当 Channel 已经注册到 Selector 后，会调用 NioEventLoop#processSelectedKey() 对不同 IO 事件进行轮询处理。<br />
　　调用 unsafe.read() 检测到 accpet 或 read 事件进行处理，主要是服务端 Channel 获取客户端的连接 Channel，将其包装成 NioSocketChannel，设置客户端的连接 Channel 设置禁止 Nagle 算法。
　　如下代码中，有两个类继承了，其中一个为 [AbstractNioMessageChannel#NioMessageUnsafe#read](https://github.com/martin-1992/Netty-Notes/blob/master/%E6%96%B0%E8%BF%9E%E6%8E%A5%E7%9A%84%E6%8E%A5%E5%85%A5/%E6%A3%80%E6%B5%8B%E6%96%B0%E8%BF%9E%E6%8E%A5.md)。

```java
    if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
        // AbstractNioMessageChannel 的 read 方法，进到 AbstractNioMessageChannel
        unsafe.read();
    }
```
    
- 在该方法中的 doReadMessages() 会接收客户端的连接，然后添加到一个 buffer 中；
- 接着调用 pipeline#fireChannelRead 方法，会从头节点 head 开始往下传播，传播到 **ServerBootstrap#channelRead，会将该客户端 Channel 注册到服务端 Channel 对应的 Selector 上；**
- 当客户端连接 Channel 注册到 Selector 后，具体注册可看 register（这里客户端 Channel 注册到服务端 Selector 流程和服务端 Channel 注册到 Selector 的流程是类似的），会对该 Channel 调用 pipeline#fireChannelActive 进行读事件注册，该方法包含两个方法。
    1. fireChannelActive，继续调用下个节点的 ChannelActive 方法；
    2. readIfIsAutoRead，从尾节点开始，往前调用 read 方法，最终调用头节点的 DefaultChannelPipeline#read 方法，将该 Channel 注册向 Selector 注册对读事件感兴趣，开始读取数据。

#### [接收客户端的数据](https://github.com/martin-1992/Netty-Notes/tree/master/%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%BF%9E%E6%8E%A5%20SocketChannel%20%E6%8E%A5%E6%94%B6%E6%95%B0%E6%8D%AE)
　　前面提到，unsafe.read() 有两个继承类实现，另一个为 AbstractNioByteChannel#NioByteUnsafe#read。

- 获取 ByteBuf；
- doReadBytes(byteBuf)，将数据写到 ByteBuf 中；
- pipeline.fireChannelRead(byteBuf)，pipeline 上执行，业务逻辑的处理在这个地方，将读到的数据进行传递，同样是从 headContext 开始往下传播；
- 读取完，调用 pipeline.fireChannelReadComplete() 传播到 pipeline 上的下个节点。

#### [业务处理](https://github.com/martin-1992/Netty-Notes/tree/master/%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%BF%9E%E6%8E%A5%20SocketChannel%20%E4%B8%9A%E5%8A%A1%E5%A4%84%E7%90%86)
　　前面提到，**会调用 AbstractNioByteChannel#NioByteUnsafe#read 接收数据，然后调用 pipeline.fireChannelRead(byteBuf) 方法，通过 pipeline 传播 ByteBuf 接收到的数据，进行业务处理。** <br />
　　对于读事件的数据，存到 ByteBuf 中，然后从 pipeline 的头结点 headContext 开始传播，经过多个 Handler 进行处理。**业务处理的本质，是数据在 pipeline 上经过所有的 Handler，调用 ChannelRead() 进行处理。**

#### [发送数据](https://github.com/martin-1992/Netty-Notes/tree/master/Netty%20%E5%8F%91%E9%80%81%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A8%8B)

- AbstractNioByteChannel#NioByteUnsafe#read 接收数据，然后调用 pipeline.fireChannelRead(byteBuf) 方法，通过 pipeline 传播 ByteBuf 接收到的数据，进行业务处理；
- 当处理完数据后，在 AbstractNioByteChannel#NioByteUnsafe#read 会调用 pipeline.fireChannelReadComplete()，示例代码中重写了 fireChannelReadComplete() 方法，会将数据发送出去，调用的是 ctx.flush()。

### [pipeline 解析](https://github.com/martin-1992/Netty-Notes/tree/master/pipeline%20%E8%A7%A3%E6%9E%90)
　　在实例化 NioServerSocketChannel 时会创建 pipeline，使用哨兵模式，默认添加两个节点 HeadContext 和 TailContext。

- 添加 ChannelHandler；

### [Netty 解码](https://github.com/martin-1992/Netty-Notes/tree/master/Netty%20%E8%A7%A3%E7%A0%81)
　　Netty 解码是将一串二进制流解码成多个 ByteBuf（自定义协议数据包），然后交给业务逻辑进行处理。

### [Recycler](https://github.com/martin-1992/Netty-Notes/tree/master/Recycler)
　　Recycler 为 Netty 的轻量级对象池，用于获取对象和用完回收对象。

### [FastThreadLocal](https://github.com/martin-1992/Netty-Notes/tree/master/FastThreadLocal)
　　Netty 中 FastThreadLocal 用来代替 ThreadLocal 存放线程本地变量，FastThreadLocal 相比 ThreadLocal 有两个优势。

- 原先的 ThreadLoacal 使用弱引用的 ThreadLoacalMap，存在内存泄漏的可能。Netty 不使用弱引用的 ThreadLocalMap，不存在该问题；
- FastThreadLocal 使用数组来存放变量，原生的 ThreadLocal 底层也是数组形式，但是使用哈希加线性探测法来实现的， 当数据量大时或遇到哈希冲突时，ThreadLocal 没有 FastThreadLocal 快。

### [时间轮 HashedWheelTimer](https://github.com/martin-1992/Netty-Notes/tree/master/%E6%97%B6%E9%97%B4%E8%BD%AE%20HashedWheelTimer)
　　时间轮是存储定时任务的环形队列，其底层为 HashedWheelBucket 数组，数组中的每个值 HashedWheelBucket 为一个双向链表，HashedWheelBucket 为定时任务的包装类，如下图。

![avatar](./时间轮%20HashedWheelTimer/photo_1.png)
