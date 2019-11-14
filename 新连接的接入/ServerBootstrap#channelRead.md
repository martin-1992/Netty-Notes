
### ServerBootstrap#channelRead
　　配置客户端的连接（NioSocketChannel），调用 [register](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/register.md) 将其注册到对应的 Selector 上，其注册流程和服务端 Channel 注册到 Selector 的流程是一致的。

- 将服务端读取到的连接（NioSocketChannel）转为 Channel;
- 在服务端启动过程，init(channel) 方法中添加用户自定义的 childHandler，该方法会回调用户重写的 ChannelInitializer#initChannel 方法，将自定义的 ChannelHandler 添加至 NioSocketChannel 的 pipeline，并在完成时删除 ChannelInitializer 自身；
- 设置客户端 Channel 的一些配置；
- 调用 [childGroup#register](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/register.md) 将客户端 Channel 注册到 worker 的 NioEventLoopGroup 的其中一个 NioEventLoop（Selector）上，**这里的 childGroup 为 worker 的 NioEventLoopGroup，注册流程和服务端 Channel 注册到 boss 的 NioEventLoopGroup 的其中一个 NioEventLoop（Selector）上是类似的。**
    1. 先判断是否在 EventLoop 线程中，这里是 false，所以会放在任务队列中执行；
    2. 调用 doRegister()，将服务端接收到的客户端 Channel 注册到 worker 的 NioEventLoop（Selector） 上；
    3. 调用 pipeline.invokeHandlerAddedIfNeeded()，回调 handlerAdded 方法，最终会调用 initChannel(Channel) 方法，该方法是用户重写的，添加用户自定义的 ChannelHandler 到客户端的 pipeline 中；
    4. 调用 pipeline.fireChannelRegistered()，回调 channelRegistered 方法；
    5. 调用 pipeline.fireChannelActive()，为客户端 Channel 注册读事件；
    5. 如果不是第一次注册，且设置自动读，为开始读取该客户端 Channel 的数据，通过 pipeline 进行传播处理。

```java
    ServerBootstrapAcceptor(
            final Channel channel, EventLoopGroup childGroup, ChannelHandler childHandler,
            Entry<ChannelOption<?>, Object>[] childOptions, Entry<AttributeKey<?>, Object>[] childAttrs) {
        this.childGroup = childGroup;
        this.childHandler = childHandler;
        this.childOptions = childOptions;
        this.childAttrs = childAttrs;
        
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
        // 在服务端启动过程，init(channel) 方法中添加用户自定义的 childHandler，该方法会
        // 回调用户重写的 ChannelInitializer#initChannel 方法，将自定义的 ChannelHandler
        // 添加至 NioSocketChannel 的 pipeline，并在完成时删除 ChannelInitializer 自身
        child.pipeline().addLast(childHandler);
        // 设置 options，跟 TCP 相关的参数
        setChannelOptions(child, childOptions, logger);
        // 设置客户端 Channel 的属性 attrs
        for (Entry<AttributeKey<?>, Object> e: childAttrs) {
            child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
        }

        try {
            // childGroup 为 worker 的 NioEventLoopGroup，调用 register 方法会返回
            // 一个 NioEventLoop，并将该客户端 Channel 注册到对应的 Selector 上
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
