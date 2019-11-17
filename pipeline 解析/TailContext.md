### DefaultChannelPipeline#TailContext
　　属于 pipeline 的一个节点，传播 Inbound 事件，父类为 AbstractChannelHandlerContext。每条消息会从头节点开始处理（调用 channelRead），往下传播，直到尾节点。如果该条消息没处理，尾节点的 channelRead 方法会尝试释放这条消息（的 buffer）。

```java
    final class TailContext extends AbstractChannelHandlerContext implements ChannelInboundHandler {

        TailContext(DefaultChannelPipeline pipeline) {
            // 属于 Inbound 的处理器，因为 inbound 为 true，outbound 为 false
            super(pipeline, null, TAIL_NAME, true, false);
            // 使用 CAS，把当前节点设置为已添加
            setAddComplete();
        }
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


### AbstractChannelHandlerContext#setAddComplete
　　使用 CAS，把当前节点设置为已添加。

```java
    private static final int INIT = 0;

    private volatile int handlerState = INIT;

    private static final int ADD_COMPLETE = 2;

    private static final int REMOVE_COMPLETE = 3;

    final boolean setAddComplete() {
        for (;;) {
            int oldState = handlerState;
            if (oldState == REMOVE_COMPLETE) {
                return false;
            }
            if (HANDLER_STATE_UPDATER.compareAndSet(this, oldState, ADD_COMPLETE)) {
                return true;
            }
        }
    }
```


### DefaultChannelPipeline#handler 
　　TailContext 是节点，也是业务逻辑处理器 ChannelHandler。每条消息会从头节点开始处理（调用 channelRead），往下传播，直到尾节点。如果该条消息没处理，尾节点的 channelRead 方法会尝试释放这条消息（的 buffer）。

```java
    @Override
    public ChannelHandler handler() {
        return this;
    }
```

### AbstractChannelHandlerContext#exceptionCaught
　　打印异常信息，并释放。

```java
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        onUnhandledInboundException(cause);
    }

    protected void onUnhandledInboundException(Throwable cause) {
        try {
            logger.warn(
                    "An exceptionCaught() event was fired, and it reached at the tail of the pipeline. " +
                            "It usually means the last handler in the pipeline did not handle the exception.",
                    cause);
        } finally {
            ReferenceCountUtil.release(cause);
        }
    }
```


### DefaultChannelPipeline#channelRead
　　一条消息从头节点传到尾节点，如果在前面的节点都没处理，传到尾节点会输出警告信息，并将这条信息释放，防止内存泄漏。

```java
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        onUnhandledInboundMessage(msg);
    }
    
    protected void onUnhandledInboundMessage(Object msg) {
        try {
            logger.debug(
                    "Discarded inbound message {} that reached at the tail of the pipeline. " +
                            "Please check your pipeline configuration.", msg);
        } finally {
            // 如果消息为可释放，最终通过内存方式释放掉，防止大量消息没处理，导致内存泄漏
            ReferenceCountUtil.release(msg);
        }
    }
```



