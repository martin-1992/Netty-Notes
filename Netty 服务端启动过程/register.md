### config().group().register(channel)
　　调用 JDK 底层来注册，将 Channel 注册到 EventLoop（事件轮询器 Selector） 上。<br />
　　调用顺序为：AbstractBootstrap#register -> SingleThreadEventLoop#register -> SingleThreadEventLoop#register -> AbstractChannel#register -> AbstractChannel#register0 -> AbstractNioChannel#doRegister（子类实现的 doRegister 方法）。

### SingleThreadEventLoop#register
　　this 为该类 SingleThreadEventLoop 的实例，而该类继承了 SingleThreadEventExecutor，为线程。将 Channel 和执行器 EventExecutor 包装成 DefaultChannelPromise。

```java
public abstract class SingleThreadEventLoop extends SingleThreadEventExecutor implements EventLoop {
    
    @Override
    public ChannelFuture register(Channel channel) {
        return register(new DefaultChannelPromise(channel, this));
    }
```

### SingleThreadEventLoop#register
　　将 DefaultChannelPromise 注册到 Selector，它是一个线程 EventLoop，this 为 SingleThreadEventLoop 类的实例。

```java
    @Override
    public ChannelFuture register(final ChannelPromise promise) {
        ObjectUtil.checkNotNull(promise, "promise");
        promise.channel().unsafe().register(this, promise);
        return promise;
    }
```

### AbstractChannel#register

- 检查是否已经注册到线程 EventLoop（Selector），是则抛出异常；
- 检查 EventLoop 的类型。类型不对，抛出异常。比如 Nio 的，则 eventLoop 类型需为 NioEventLoop；
- 注册 EventLoop，后续 Channel 的操作，都是由 EventLoop 来处理；
- 判断当前线程是否为 EventLoop，只有在 EventLoop 线程才可进行操作；
- register0，注册的核心逻辑；
- 如果当前线程不为 EventLoop，则创建一个新线程调用 register0。

```java
    private volatile EventLoop eventLoop;

    @Override
    public final void register(EventLoop eventLoop, final ChannelPromise promise) {
        if (eventLoop == null) {
            throw new NullPointerException("eventLoop");
        }
        // 已经注册过，抛出异常
        if (isRegistered()) {
            promise.setFailure(new IllegalStateException("registered to an event loop already"));
            return;
        }
        // 类型不对，比如 Nio 的，则 eventLoop 类型需为 NioEventLoop
        if (!isCompatible(eventLoop)) {
            promise.setFailure(
                    new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
            return;
        }
        // 注册 EventLoop（Selector），后续 IO 操作都由 eventLoop 来处理
        AbstractChannel.this.eventLoop = eventLoop;
        
        // 判断当前线程是否为 EventLoop，只有在 EventLoop 线程才可进行操作 
        if (eventLoop.inEventLoop()) {
            // 实际注册在这里
            register0(promise);
        } else {
            try {
                // 如果当前线程不为 EventLoop，则创建一个新线程调用 register0
                eventLoop.execute(new Runnable() {
                    @Override
                    public void run() {
                        register0(promise);
                    }
                });
            } catch (Throwable t) {
                logger.warn(
                        "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                        AbstractChannel.this, t);
                closeForcibly();
                closeFuture.setClosed();
                safeSetFailure(promise, t);
            }
        }
    }
```

### AbstractChannel#register0

- 调用 doRegister() 方法，将服务端 Channel 注册到 boss 的 NioEventLoopGroup 的其中一个 NioEventLoop（Selector） 上。doRegister() 方法在 AbstractChannel#doRegister 实现为空，交由子类来实现，这里以 AbstractNioChannel#doRegister 为例。之后，服务端 Channel 会注册读事件，然后通过 pipeline 在节点 ServerBootstrap 调用 channelRead 方法，将读事件服务端接收到客户端 Channel 的连接。同样是调用该方法，不过是将客户端 Channel 注册到 worker 的 NioEventLoopGroup 的其中一个 NioEventLoop（Selector） 上；
- 调用 pipeline.invokeHandlerAddedIfNeeded()，回调 handlerAdded 方法，最终会调用 initChannel(Channel) 方法，该方法是用户重写的，添加用户自定义的 ChannelHandler 到 pipeline 中；
- 当新连接注册到 EventLoop（Selector） 上，第一次注册时会调用 [pipeline.fireChannelActive]() 将读事件注册到 Selector 上。当有客户端连接进来时，会进行处理。

```java
        private void register0(ChannelPromise promise) {
            try {
                if (!promise.setUncancellable() || !ensureOpen(promise)) {
                    return;
                }
                boolean firstRegistration = neverRegistered;
                //  将 Channel 注册到 EventLoop（Selector） 上
                doRegister();
                // 从 true 变为 false，表示已经注册过了
                neverRegistered = false;
                registered = true;

                // 回调 handlerAdded 方法，最终会调用 initChannel(Channel) 方法，该方法
                // 是用户重写的，添加用户自定义的 ChannelHandler 到 pipeline 中
                pipeline.invokeHandlerAddedIfNeeded();

                safeSetSuccess(promise);
                // 回调 channelRegistered 方法
                pipeline.fireChannelRegistered();
                // 新连接，并注册到 EventLoop（Selector） 上，则为 true
                if (isActive()) {
                    if (firstRegistration) {
                        // 第一次注册，读事件绑定
                        pipeline.fireChannelActive();
                    } else if (config().isAutoRead()) {
                        // 读数据
                        beginRead();
                    }
                }
            } catch (Throwable t) {
                // Close the channel directly to avoid FD leak.
                closeForcibly();
                closeFuture.setClosed();
                safeSetFailure(promise, t);
            }
        }
```

### AbstractNioChannel#doRegister
　　通过自旋，调用 JDK 底层来注册，保证 Channel 注册到 EventLoop（Selector） 上，这里注册的 this 指的是 AbstractNioChannel，将其作为 SelectableChannel 类的附属 attachment，注册到 Selector 上。在 [processSelectedKeys](https://github.com/martin-1992/Netty-Notes/blob/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%90%AF%E5%8A%A8/processSelectedKeys.md) 方法中，Selector 轮询 SelectionKey 时，会取出该 key 的 attachment，即 AbstractNioChannel，检查该 AbstractNioChannel 是否有 IO 事件要处理。

```java
    @Override
    protected void doRegister() throws Exception {
        boolean selected = false;
        // 自旋过程，保证注册成功，失败则抛出异常
        for (;;) {
            try {
                // 调用 JDK 底层来注册，channel 是通过 AbstractBootstrap#this.channelFactory.newChannel() 
                // 进行创建的，将 Channel 注册到 Selector 上，这里注册为 0，表示对任何事件感兴趣，接下来会
                // 调用 pipeline.fireChannelActive() 进行读事件注册
                selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    throw e;
                }
            }
        }
    }
```
