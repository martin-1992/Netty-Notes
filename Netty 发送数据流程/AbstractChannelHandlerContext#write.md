### AbstractChannelHandlerContext#write

- 找到下个 outBound（headContext），选择该 outBound 绑定的 NioEventLoop；
- 如果是 flush，则调用 [invokeWriteAndFlush]() 方法。否则调用 [AbstractChannelHandlerContext#invokeWrite]() 方法。

```java
    @Override
    public ChannelFuture write(Object msg) {
        return write(msg, newPromise());
    }
    
    @Override
    public ChannelFuture write(final Object msg, final ChannelPromise promise) {
        write(msg, false, promise);

        return promise;
    }
    
    private void write(Object msg, boolean flush, ChannelPromise promise) {
        ObjectUtil.checkNotNull(msg, "msg");
        try {
            // 回调函数是已取消的，则释放消息
            if (isNotValidPromise(promise, true)) {
                ReferenceCountUtil.release(msg);
                return;
            }
        } catch (RuntimeException e) {
            // 出现问题，释放消息，防止内存泄漏
            ReferenceCountUtil.release(msg);
            throw e;
        }
        // 找到下个 outBound
        AbstractChannelHandlerContext next = findContextOutbound();
        // 引用计数用的，用来检测内存泄漏
        final Object m = pipeline.touch(msg, next);
        // EventExecutor 数组（NioEventLoop 数组），选择一个 NioEventLoop 来执行
        EventExecutor executor = next.executor();
        // 判断是否在 EventLoop 线程中，调用下个节点的 invokeWrite 或 invokeWriteAndFlush 方法，不断往前传播
        if (executor.inEventLoop()) {
            if (flush) {
                // 写和刷新，对应 writeAndFlush 方法
                next.invokeWriteAndFlush(m, promise);
            } else {
                // 只写，对应 write 方法
                next.invokeWrite(m, promise);
            }
        } else {
            // 不在 EventLoop，则放到任务队列执行
            final AbstractWriteTask task;
            if (flush) {
                task = WriteAndFlushTask.newInstance(next, m, promise);
            }  else {
                task = WriteTask.newInstance(next, m, promise);
            }
            if (!safeExecute(executor, task, promise, m)) {
                task.cancel();
            }
        }
    }
```

#### AbstractChannelHandlerContext#findContextOutbound
　　oubBound 是从后往前的，即从 tail -> head，所以这里是往前找下个 handlerContext。

```java
    private AbstractChannelHandlerContext findContextOutbound() {
        AbstractChannelHandlerContext ctx = this;
        // 通过 while 循环找到前面的节点
        do {
            ctx = ctx.prev;
        } while (!ctx.outbound);
        return ctx;
    }
```

### AbstractChannelHandlerContext#invokeWrite
　　前面提到会找到下个 outBound，然后通过反射代用其 write 方法。以 Netty 示例代码中的 EchoServerHandler 为例，其添加顺序为 HeadContext <- EchoServerHandler <- TailContext。<br />
　　Netty 是在 EchoServerHandler#channelRead()，调用 ctx.write(msg)。所以往前调用下个 outBound 为 HeadContext，即调用 HeadContext 的 write 方法。

```java
    private void invokeWrite(Object msg, ChannelPromise promise) {
        if (invokeHandler()) {
            invokeWrite0(msg, promise);
        } else {
            write(msg, promise);
        }
    }
    
    private void invokeWrite0(Object msg, ChannelPromise promise) {
        try {
            // 拿到节点 Handler，并转为 ChannelOutboundHandler，调用 write 方法
            // 这个 write 方法是回调方法，会回调示例代码中的自定义 write 函数，
            // 即代码 System.out.println("OutBoundHandlerC: " + msg)，打印输出
            // 信息，然后往下执行 ctx.write，会继续向前传播
            ((ChannelOutboundHandler) handler()).write(this, msg, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise);
        }
    }
```

### DefaultChannelPipeline#HeadContext#write
　　写操作使用 unsafe，封装后调用。

```java
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        unsafe.write(msg, promise);
    }
```