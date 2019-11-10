### DefaultChannelPipeline#HeadContext
　　通过封装 unsafe，调用它的方法来进行数据的读写。

- 读、写、连接等操作，都是通过封装 unsafe，调用它的方法实现；
- channelActive，连接建立成功后，调用该方法，自动注册读事件并传播，从 HeadContext 开始。

```java
    final class HeadContext extends AbstractChannelHandlerContext
            implements ChannelOutboundHandler, ChannelInboundHandler {
        // 用于底层数据的读写
        private final Unsafe unsafe;

        HeadContext(DefaultChannelPipeline pipeline) {
            super(pipeline, null, HEAD_NAME, true, true);
            // 获取 channel 的 unsafe
            unsafe = pipeline.channel().unsafe();
            // 标识节点已被添加
            setAddComplete();
        }
        
    AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name,
                                  boolean inbound, boolean outbound) {
        this.name = ObjectUtil.checkNotNull(name, "name");
        this.pipeline = pipeline;
        this.executor = executor;
        this.inbound = inbound;
        this.outbound = outbound;
        // Its ordered if its driven by the EventLoop or the given Executor is an instanceof OrderedEventExecutor.
        ordered = executor == null || executor instanceof OrderedEventExecutor;
    }
```

### DefaultChannelPipeline#handler 
　　HeadContext 是节点，也是业务逻辑处理器 ChannelHandler。

```java
    @Override
    public ChannelHandler handler() {
        return this;
    }
```

### DefaultChannelPipeline#bind
　　调用 JDK 底层绑定端口。

```java
    @Override
    public void bind(
            ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) {
        unsafe.bind(localAddress, promise);
    }
```

#### AbstractChannel#bind
　　绑定端口的逻辑在 doBind，当端口绑定成功后，会调用当前 EventLoop 线程创建一个传播任务放到任务队列中执行。

```java
    @Override
    public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
        // 判断当前线程是否在 EventLoop 线程中
        assertEventLoop();

        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }

        // See: https://github.com/netty/netty/issues/576
        if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
            localAddress instanceof InetSocketAddress &&
            !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
            !PlatformDependent.isWindows() && !PlatformDependent.maybeSuperUser()) {
            // Warn a user about the fact that a non-root user can't receive a
            // broadcast packet on *nix if the socket is bound on non-wildcard address.
            logger.warn(
                    "A non-root user can't receive a broadcast packet if the socket " +
                    "is not bound to a wildcard address; binding to a non-wildcard " +
                    "address (" + localAddress + ") anyway as requested.");
        }
        // 端口绑定未完成，这里为 false，即在服务端启动之前
        boolean wasActive = isActive();
        try {
            // 绑定端口
            doBind(localAddress);
        } catch (Throwable t) {
            safeSetFailure(promise, t);
            closeIfClosed();
            return;
        }
        // wasActive=false，端口绑定前
        // isActive=true，端口绑定后
        if (!wasActive && isActive()) {
            invokeLater(new Runnable() {
                @Override
                public void run() {
                    // 传播 Active 事件
                    pipeline.fireChannelActive();
                }
            });
        }

        safeSetSuccess(promise);
    }
```

#### NioServerSocketChannel#doBind
　　根据版本号，调用 JDK 底层 API 进行端口绑定。

```java
    @Override
    protected void doBind(SocketAddress localAddress) throws Exception {
        if (PlatformDependent.javaVersion() >= 7) {
            javaChannel().bind(localAddress, config.getBacklog());
        } else {
            javaChannel().socket().bind(localAddress, config.getBacklog());
        }
    }
```

### DefaultChannelPipeline#handler 
　　读取数据

```java
    @Override
    public void read(ChannelHandlerContext ctx) {
        // 调用 AbstractChannel 的，读取数据
        unsafe.beginRead();
    }
        
    /**
     * AbstractChannel#beginRead
     */
    @Override
    public final void beginRead() {
        assertEventLoop();

        if (!isActive()) {
            return;
        }
        try {
            // 核心逻辑
            doBeginRead();
        } catch (final Exception e) {
            invokeLater(new Runnable() {
                @Override
                public void run() {
                    pipeline.fireExceptionCaught(e);
                }
            });
            close(voidPromise());
        }
    }
```

#### doBeginRead

- 获取感兴趣的 IO 事件集合；
- 如果集合中不包含读事件，则将读事件注册到 selectionKey，这样就可以处理读事件。

```java
    @Override
    protected void doBeginRead() throws Exception {
        // 获取 selectionKey
        final SelectionKey selectionKey = this.selectionKey;
        if (!selectionKey.isValid()) {
            return;
        }

        readPending = true;
        
        // 获取感兴趣的 IO 事件集合
        final int interestOps = selectionKey.interestOps();
        // 如果集合中不包含读事件，则将读事件注册到 selectionKey
        if ((interestOps & readInterestOp) == 0) {
            selectionKey.interestOps(interestOps | readInterestOp);
        }
    }
```

### DefaultChannelPipeline#connect
　　同样是调用 JDK 底层 API 的方法。

```java
    @Override
    public void connect(
            ChannelHandlerContext ctx,
            SocketAddress remoteAddress, SocketAddress localAddress,
            ChannelPromise promise) {
        unsafe.connect(remoteAddress, localAddress, promise);
    }

            @Override
    public void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) {
        unsafe.disconnect(promise);
    }

    @Override
    public void close(ChannelHandlerContext ctx, ChannelPromise promise) {
        unsafe.close(promise);
    }

    @Override
    public void deregister(ChannelHandlerContext ctx, ChannelPromise promise) {
        unsafe.deregister(promise);
    }
```
