### DefaultChannelPipeline#newContext

- 构建节点前，会判断该节点的名字是否重复，通过遍历方式，看是否有其他 ChannelHandler 使用；
- 调用 AbstractChannelHandlerContext 的构造函数来构建节点，使用变量保存 handler、pipeline、executor、inbound、outbound；

```java
    // 创建 ChannelHandlerContext 节点，调用上一节提到的 AbstractChannelHandlerContext
    private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
        return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
    }
    
    /**
     * DefaultChannelHandlerContext 的构造函数
     */
    DefaultChannelHandlerContext(
            DefaultChannelPipeline pipeline, EventExecutor executor, String name, ChannelHandler handler) {
        super(pipeline, executor, name, isInbound(handler), isOutbound(handler));
        if (handler == null) {
            throw new NullPointerException("handler");
        }
        this.handler = handler;
    }
```


### AbstractChannelHandlerContext#AbstractChannelHandlerContext
　　AbstractChannelHandlerContext 的构造函数，保存成员变量，包括 pipeline、executor、inbound、outbound 等。

```java
    AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name,
                                  boolean inbound, boolean outbound) {
        // 节点 ChannelHandlerContext 的名字
        this.name = ObjectUtil.checkNotNull(name, "name");
        this.pipeline = pipeline;
        this.executor = executor;
        this.inbound = inbound;
        this.outbound = outbound;
        // Its ordered if its driven by the EventLoop or the given Executor is an instanceof OrderedEventExecutor.
        ordered = executor == null || executor instanceof OrderedEventExecutor;
    }
```

