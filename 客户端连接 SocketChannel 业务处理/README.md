### AbstractNioByteChannel#read
　　在 [客户端 SocketChannel 接收数据中](https://github.com/martin-1992/Netty-Notes/tree/master/%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%BF%9E%E6%8E%A5%20SocketChannel%20%E6%8E%A5%E6%94%B6%E6%95%B0%E6%8D%AE) ，会调用 pipeline.fireChannelRead(byteBuf) 方法，通过 pipeline 传播 ByteBuf 接收到的数据，进行业务处理。<br />
　　对于读事件的数据，存到 ByteBuf 中，然后从 pipeline 的头结点 headContext 开始传播，经过多个 Handler 进行处理。**业务处理的本质，是数据在 pipeline 上经过所有的 Handler，调用 ChannelRead() 进行处理。**

```java
    @Override
    public final void read() {
        // ...
        try {
            do {
                // ...
                // 记录读取的次数 + 1
                allocHandle.incMessagesRead(1);
                readPending = false;
                // pipeline 上执行，业务逻辑的处理在这个地方，将读到的数据进行传递，同样
                // 是从 headContext 开始往下传播
                pipeline.fireChannelRead(byteBuf);
                byteBuf = null;
                // 读取一次，就跳出循环
            } while (allocHandle.continueReading());
            // ...
```

### DefaultChannelPipeline#fireChannelRead
　　pipeline 传播，从 headContext 开始。

```java
    @Override
    public final ChannelPipeline fireChannelRead(Object msg) {
        // 从 head 开始向下传播
        AbstractChannelHandlerContext.invokeChannelRead(head, msg);
        return this;
    }
```

### AbstractChannelHandlerContext#invokeChannelRead
　　调用节点 Handler 的 ChannelRead() 方法。

```java
    static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
        final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
        // 获取该 Channel 的 EventLoop（NioEventLoop）
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            // 如在 EventLoop 线程中，则调用该方法 invokeChannelRead，
            // 否则另外起一个线程来运行该任务，保证线程安全。
            // 第一次运行 next 为 HeadContext，即从头节点开始往下传播，
            // 调用头节点的 invokeChannelRead 方法，next 为 header
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

### DefaultChannelPipeline#channelRead
　　把当前定义的事件（比如接收到数据的 ByteBuf）传播中的对象通过 fireChannelRead 进行传播，ctx 为在 pipeline 上继续传播。

```java
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ctx.fireChannelRead(msg);
    }
```

### AbstractChannelHandlerContext#fireChannelRead
　　findContextInbound，寻找下一个 InboundHandler，找到后通过 invokeChannelRead 把事件继续往下传播。

```java
    @Override
    public ChannelHandlerContext fireChannelRead(final Object msg) {
        invokeChannelRead(findContextInbound(), msg);
        return this;
    }
    
    private AbstractChannelHandlerContext findContextInbound() {
        AbstractChannelHandlerContext ctx = this;
        do {
            // 找到下个 handler
            ctx = ctx.next;
        } while (!ctx.inbound);
        return ctx;
    }
```
