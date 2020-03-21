### DefaultChannelPipeline#fireChannelRead
　　调用 pipeline.fireChannelRead()，从 headContext 开始往下传播。然后传到 ServerBootstrap#ServerBootstrapAcceptor，调用 ServerBootstrapAcceptor.fireChannel()，将客户端 Channel 注册到 workerGroup 上。<br />
　　前面讲到在 [Netty 服务端启动过程中会初始化 Channel](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/init.md)，进行配置，包括为该服务端 Channel 的 pipeline 添加 ServerBootstrapAcceptor。

```java
    @Override
    public final ChannelPipeline fireChannelRead(Object msg) {
        // 从 head 开始向下传播
        AbstractChannelHandlerContext.invokeChannelRead(head, msg);
        return this;
    }
```

### AbstractChannelHandlerContext#invokeChannelRead
　　往下继续调用。

```java
    static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
        final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            // 如在 EventLoop 线程中，则调用该方法 invokeChannelRead，
            // 否则另外起一个线程来运行该任务，保证线程安全。
            // 第一次运行 next 为 HeadContext，即从头节点开始往下传播，
            // 调用头节点的 invokeChannelRead 方法
            next.invokeChannelRead(m);
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelRead(m);
                }
            });
        }
    }
    
    @Override
    public EventExecutor executor() {
        if (executor == null) {
            return channel().eventLoop();
        } else {
            return executor;
        }
    }
```

### AbstractChannelHandlerContext#invokeChannelRead
　　调用服务端 Channel 的 pipeline 中下个节点 Context。

```java
    private void invokeChannelRead(Object msg) {
        if (invokeHandler()) {
            try {
                // 调用 Handler 的 channelRead() 方法
                ((ChannelInboundHandler) handler()).channelRead(this, msg);
            } catch (Throwable t) {
                // 出现异常时调用，异常的传播
                notifyHandlerException(t);
            }
        } else {
            fireChannelRead(msg);
        }
    }
```

### ServerBootstrap#channelRead
　　前面讲到在 [Netty 服务端启动过程中会初始化 Channel](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/init.md)，进行配置，包括为该服务端 Channel 的 pipeline 添加 ServerBootstrapAcceptor。
　　这里调用 ServerBootstrapAcceptor#channelRead() 为客户端 Channel 配置属性，并注册到 workerGroup 中的一个 NioEventLoop 上。同 bossGroup 一样，childGroup 为一个 EventLoopGroup，即 NioEventLoop 的数组。



### ServerBootstrap#channelRead
　　前面讲到在 [Netty 服务端启动过程中会初始化 Channel](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/init.md)，进行配置，包括为该服务端 Channel 的 pipeline 添加 ServerBootstrapAcceptor。
　　这里调用 ServerBootstrapAcceptor#channelRead() 为客户端 Channel 配置属性，调用 [register](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/register.md) 注册到 workerGroup 中的一个 NioEventLoop 上。同 bossGroup 一样，childGroup 为一个 EventLoopGroup，即 NioEventLoop 的数组，其注册流程和服务端 Channel 注册到 Selector 的流程是一致的。

- 为客户端 Channel 配置属性，包括添加用户自定义的 childHandler、设置跟 TCP 相关的参数、设置客户端 Channel 的属性 attrs；
    1. 在服务端启动过程，init(channel) 方法中添加用户自定义的 childHandler，该方法会回调用户重写的 ChannelInitializer#initChannel 方法，将自定义的 ChannelHandler 添加至 NioSocketChannel 的 pipeline，并在完成时删除 ChannelInitializer 自身；
    2. 设置客户端 Channel 的一些配置。
- 调用 [childGroup#register](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/register.md) 将客户端 Channel 注册到 workerGroup 的 NioEventLoopGroup 的其中一个 NioEventLoop（Selector）上，**这里的 childGroup 为 workerGroup 的 NioEventLoopGroup，注册流程和服务端 Channel 注册到 boss 的 NioEventLoopGroup 的其中一个 NioEventLoop（Selector）上是类似的。**
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
