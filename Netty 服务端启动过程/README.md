### Netty 服务端启动过程
　　这里使用一个简单例子来说明，在配置好 EventLoopGroup、Channel、ChannelHandler 等等，绑定端口最终会调用 dobind() 方法。

```java
public final class Server {
    
    public static void main(String[] args) throws Exception {
        // 主从 Reactor，bossGroup 为 mainReactor，workerGroup 为 subReactor
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            // 设置引导类
            ServerBootstrap b = new ServerBootstrap();
            // 通过 group 将两大线程配置进来
            b.group(bossGroup, workerGroup)
                    // 设置服务端的 ServerSocketChannel
                    .channel(NioServerSocketChannel.class)
                    // 给后面指定的每个客户端的连接，设置 TCP 的基本属性
                    .childOption(ChannelOption.TCP_NODELAY, true)
                    // 每次创建客户端连接时，绑定一些基本的属性
                    .childAttr(AttributeKey.newInstance("childAttr"), "childAttrValue")
                    // 服务端启动过程中的处理逻辑
                    .handler(new ServerHandler())
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) {
                            ch.pipeline().addLast(new AuthHandler());
                            //..

                        }
                    });
            // 绑定端口，最终调用 dobind() 方法
            ChannelFuture f = b.bind(8888).sync();

            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

- 在 mainReactor，即 bossGroup 的 NioEventLoopGroup 构造函数中，调用 openSelector 创建 Selector。一个线程，绑定一个 Selector，为客户端连接做准备；
- 创建服务端 Channel、初始化配置服务端 Channel；
- 将服务端 Channel 注册到 Selector 上，即从 bossGroup 中选择一个 NioEventLoop，将 Channel 注册到该 NioEventLoop 上。注意，示例代码中 bossGroup 的线程为 1，new NioEventLoopGroup(1)，所以只有一个 NioEventLoop。在多线程的主从 Reactor 中，bossGroup 会有多个 NioEventLoop；
- 将 Channel 注册到 Selector 上，获取 selectionKey。设置 ops 为 0，即对任何事件都不感兴趣；
- 绑定端口；
- 绑定成功后，isActive 为 true。会使用 pipeline，调用 pipeline.fireChannelAcitve。从 headContext 开始设置服务端 Channel 的 ops 为 OP_ACCEPT，完成服务端连接准备。之后客户端连接进来，会为其创建连接和读取数据。

![avatar](photo_1.png)

### NioEventLoopGroup 主要作用

- 创建 EventExecutor 数组（NioEventLoop 数组），初始化 NioEventLoop 放入 EventExecutor 数组；
- 将初始化好后的 EventExecutor 使用 chooser 变量保存，这样可通过 next() 方法获取 NioEventLoop，然后服务端连接 channel 选择一个 NioEventLoop 进行注册，后续对 Channel 的操作通过 NioEventLoop 进行。

### NioEventLoop 主要作用

- 创建一个优化后的创建一个 Selector（IO 事件轮询器）；
	1. 一个 Selector 绑定一个线程 NioEventLoop；
	2. 一个 Selector 下有多个 Channel（包装的 Socket）；
	3. 这里会对 Selector 的 keySet 进行优化，从 hashset 转换为数组，便于 Netty 遍历。
- 创建线程工厂 Executor；
- 创建任务队列 taskQueue；
- 设置了队列的拒绝策略。

### AbstractBootstrap#doBind
　　以服务端的 NioEventLoopGroup 为例，在[NioEventLoopGroup 的构造函数中](https://github.com/martin-1992/Netty-Notes/tree/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%88%9B%E5%BB%BA)，会创建 nthreads 个服务端的 Selecotr（这里为 nthreads 为 1，因为 new NioEventLoopGroup(1)）。然后在调用绑定端口时，会创建 ServerSocketChannel，初始化 ServerSocketChannel，最后将 ServerSocketChannel 注册到之前 NioEventLoopGroup 创建的 Selector 上。
- [initAndRegister](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/initAndRegister.md)，创建 Channel、初始化配置 Channel、将 Channel 注册到 EventLoop（事件轮询器 Selector）；
- [doBind0](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/doBind0.md)，调用 JDK 底层 API 将端口与 [initAndRegister](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/initAndRegister.md) 创建好的 Channel 进行绑定，并添加监听器。

```java
    private ChannelFuture doBind(final SocketAddress localAddress) {
        // 创建、初始化、注册 Channel
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }

        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            // 绑定端口
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // ...
                }
            });
            return promise;
        }
    }
```
