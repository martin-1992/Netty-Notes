### DefaultChannelPipeline#bind
　　端口绑定方法，层层调用，最后会调用 AbstractChannel#bind 方法。

```java
    @Override
    public void bind(
            ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) {
        unsafe.bind(localAddress, promise);
    }
```

### AbstractChannel#bind

- 判断当前线程是否在 EventLoop 线程中；
- isActive()，调用 JDK 底层 API 判断该端口是否绑定，初次进行为未绑定；
- doBind()，调用 JDK 底层 API 进行端口绑定，doBind 为 Channel 接口的方法，由子类实现；
- [pipeline.fireChannelActive()](https://github.com/martin-1992/Netty-Notes/blob/master/%E6%96%B0%E8%BF%9E%E6%8E%A5%E7%9A%84%E6%8E%A5%E5%85%A5/pipeline%23fireChannelActive.md)，端口绑定完成后，会通过 pipeline，从 headContext 设置服务端 Channel 的 ops 为 OP_ACCEPT，完成服务端连接准备。之后客户端连接进来，会为其创建连接和读取数据。

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
            // 调用 JDK 底层 API 进行端口绑定，doBind 为 Channel 接口的方法，由子类实现
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

### NioServerSocketChannel#doBind
　　doBind 为 Channel 接口的方法，由子类实现，这里以 NioServerSocketChannel 为例，调用 JDK 底层 API 进行端口绑定。

```java
    @Override
    protected void doBind(SocketAddress localAddress) throws Exception {
        if (PlatformDependent.javaVersion() >= 7) {
            // 调用 JDK 底层 API 进行端口绑定
            javaChannel().bind(localAddress, config.getBacklog());
        } else {
            javaChannel().socket().bind(localAddress, config.getBacklog());
        }
    }
```

### NioServerSocketChannel#isActive
　　调用 JDK 底层 API 判断该端口是否绑定。

```java
    @Override
    public boolean isActive() {
        return javaChannel().socket().isBound();
    }
```
