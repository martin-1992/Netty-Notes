### Netty 服务端启动过程
　　这里使用一个简单例子来说明，在配置好 EventLoopGroup、Channel、ChannelHandler 等等，绑定端口最终会调用 dobind() 方法。

```java
public final class Server {
    
    public static void main(String[] args) throws Exception {
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

### AbstractBootstrap#doBind

- [initAndRegister](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/initAndRegister.md)，创建 Channel、初始化配置 Channel、将 Channel 注册到 EventLoop（事件轮询器 Selector）；
- [doBind0](https://github.com/martin-1992/Netty-Notes/edit/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/doBind0.md)，调用 JDK 底层 API 进行端口绑定。

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
