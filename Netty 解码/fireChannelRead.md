### ByteToMessageDecoder#fireChannelRead
　　遍历对象列表，将每个对象作为参数传给下个节点的 ChannelRead 方法，进行业务处理。

```java
    static void fireChannelRead(ChannelHandlerContext ctx, List<Object> msgs, int numElements) {
        if (msgs instanceof CodecOutputList) {
            fireChannelRead(ctx, (CodecOutputList) msgs, numElements);
        } else {
            for (int i = 0; i < numElements; i++) {
                ctx.fireChannelRead(msgs.get(i));
            }
        }
    }
    
    static void fireChannelRead(ChannelHandlerContext ctx, CodecOutputList msgs, int numElements) {
        for (int i = 0; i < numElements; i ++) {
            ctx.fireChannelRead(msgs.getUnsafe(i));
        }
    }
```

### AbstractChannelHandlerContext#fireChannelRead

- 调用 findContextInbound()，获取下个节点 ChannelHandlerContext；
- 如果当前线程为 EventLoop 线程，调用 invokeChannelRead，该方法会调用下个节点的 ChannelRead，传入之前解码出来的对象；
- 不为 EventLoop 线程，则放到线程工厂执行。

```java
    @Override
    public ChannelHandlerContext fireChannelRead(final Object msg) {
        invokeChannelRead(findContextInbound(), msg);
        return this;
    }
    
    static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
        final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
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
```

### findContextInbound
　　获取下个节点 ChannelHandlerContext。

```java
    private AbstractChannelHandlerContext findContextInbound() {
        AbstractChannelHandlerContext ctx = this;
        do {
            ctx = ctx.next;
        } while (!ctx.inbound);
        return ctx;
    }
```
