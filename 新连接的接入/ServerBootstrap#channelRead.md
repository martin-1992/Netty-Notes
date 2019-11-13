
### ServerBootstrap#channelRead
　　配置客户端的连接（NioSocketChannel），调用 [register](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/register.md) 将其注册到对应的 Selector 上，其注册流程和服务端 Channel 注册到 Selector 的流程是一致的。


```java
    ServerBootstrapAcceptor(
            final Channel channel, EventLoopGroup childGroup, ChannelHandler childHandler,
            Entry<ChannelOption<?>, Object>[] childOptions, Entry<AttributeKey<?>, Object>[] childAttrs) {
        this.childGroup = childGroup;
        this.childHandler = childHandler;
        this.childOptions = childOptions;
        this.childAttrs = childAttrs;

        // Task which is scheduled to re-enable auto-read.
        // It's important to create this Runnable before we try to submit it as otherwise the URLClassLoader may
        // not be able to load the class because of the file limit it already reached.
        //
        // See https://github.com/netty/netty/issues/1328
        enableAutoReadTask = new Runnable() {
            @Override
            public void run() {
                channel.config().setAutoRead(true);
            }
        };
    }

    @Override
    @SuppressWarnings("unchecked")
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 将服务端读取到的连接（NioSocketChannel）转为 Channel
        final Channel child = (Channel) msg;
        // 在服务端启动过程，init(channel) 方法中添加用户自定义的 childHandler
        child.pipeline().addLast(childHandler);
        // 设置 options，跟 TCP 相关的参数
        setChannelOptions(child, childOptions, logger);
        // 设置客户端 Channel 的属性 attrs
        for (Entry<AttributeKey<?>, Object> e: childAttrs) {
            child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
        }

        try {
            // childGroup 为 NioEventLoopGroup，调用 register 方法会返回一个 NioEventLoop，
            // 并将该客户端 Channel 注册到对应的 Selector 上
            childGroup.register(child).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (!future.isSuccess()) {
                        forceClose(child, future.cause());
                    }
                }
            });
        } catch (Throwable t) {
            forceClose(child, t);
        }
    }
```

### MultithreadEventLoopGroup#register
　　调用 [register](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/register.md)，同服务端 Channel 注册到 Selector 的流程是一致的。

```java
    @Override
    public ChannelFuture register(Channel channel) {
        // next() 方法返回一个 NioEventLoop
        return next().register(channel);
    }
```

### next
　　从 NioEventLoopGroup 返回一个 NioEventLoop。

```java
    @Override
    public EventLoop next() {
        return (EventLoop) super.next();
    }

    @Override
    public EventExecutor next() {
        // chooser 每次会选择一个 NioEventLoop
        return chooser.next();
    }
```
