### AbstractBootstrap#initAndRegister
　　创建 Channel、初始化配置 Channel、将 Channel 注册到 EventLoop（事件轮询器 Selector）。

- [channelFactory.newChannel()](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/newChannel.md)，通过反射 + 泛型进行实例化，来完成创建 Channel。而创建的 Channel 是在代码配置中 .channel(NioServerSocketChannel.class) 传入的 NioServerSocketChannel.class；
    1. 实例化调用的是 ReflectiveChannelFactory 的 constructor.newInstance() 方法；
    2. 而传入的 [NioServerSocketChannel](https://github.com/martin-1992/Netty-Notes/blob/master/%E6%96%B0%E8%BF%9E%E6%8E%A5%E7%9A%84%E6%8E%A5%E5%85%A5/NioServerSocketChannel.md)，是传入到 ReflectiveChannelFactory 类的构造函数中，使用 constructor 变量记录，该类**用变量 ch 保存服务端创建的 Channel（用于为客户端连接做准备），还有 Pipeline、unsafe 等**。
- [init(channel)](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/init.md)，配置初始化服务端 Channel，通过引导类 ServerBootstrap 传入的参数进行配置。NioServerSocketChannel 类会调用父类构造函数，创建 Pipeline，并添加节点 [ServerBootstrapAcceptor]()。ServerBootstrapAcceptor 负责接收客户端连接 Socket 创建连接后，对连接的初始化；
- [config().group().register(channel)](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/register.md)，通过自旋，调用 JDK 底层来注册，保证 Channel 注册到 EventLoop（事件轮询器 Selector） 上。后续对 Channel 的操作，都由 EventLoop 来处理。

### AbstractBootstrap#initAndRegister

```java
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            // ...
        
        ChannelFuture regFuture = config().group().register(channel);
        // Channel 已注册则关闭
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }
        return regFuture;
    }
```
