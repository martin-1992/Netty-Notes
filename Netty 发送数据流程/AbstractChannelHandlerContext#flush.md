### AbstractChannelHandlerContext#flush
　　找到下一个 handler，执行发送消息流程。

```java
    @Override
    public ChannelHandlerContext flush() {
        // 找下一个 handler
        final AbstractChannelHandlerContext next = findContextOutbound();
        // 从 NioEventLoop 数组中选择一个 NioEventLoop
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            // 发送
            next.invokeFlush();
        } else {
            Tasks tasks = next.invokeTasks;
            if (tasks == null) {
                next.invokeTasks = tasks = new Tasks(next);
            }
            safeExecute(executor, tasks.invokeFlushTask, channel().voidPromise(), null);
        }

        return this;
    }

    private void invokeFlush() {
        if (invokeHandler()) {
            invokeFlush0();
        } else {
            flush();
        }
    }

    private void invokeFlush0() {
        try {
            ((ChannelOutboundHandler) handler()).flush(this);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    }
```

### DefaultChannelPipeline#HeadContext#flush
　　调用头节点的 flush 方法，前面说过是从后往前传播，当前 hanlder 的前面一个为 HeadContext。

```java
    @Override
    public void flush(ChannelHandlerContext ctx) {
        unsafe.flush();
    }
```

### AbstractChannel#flush
　　更改数据状态，由未发送 unflushedEntry 改为已发送 flushedEntry。

```java
    @Override
    public final void flush() {
        assertEventLoop();
        ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
        // 为 null，表示 Channel 已经关闭了
        if (outboundBuffer == null) {
            return;
        }
        // 将未发送的数据节点 unflushedEntry 转为已发送数据的节点 flushedEntry
        outboundBuffer.addFlush();
        // 发送出去
        flush0();
    }
```

#### addFlush
　　将未发送的数据节点 unflushedEntry 转为已发送数据的节点 flushedEntry，在 [AbstractChannel#write]() 提到 Entry 为一个节点，多个 Entry 组成一个单向链表。
                                                                                 
- tailEntry，为缓冲区 ChannelOutboundBuffer 的最后一个节点；
- unFushedEntry，链表中指向第一个没有写到底层系统缓冲区 Socket 中的节点；
- flushedEntry，链表中指向第一个写入到底层系统缓冲区 Socket 中的节点。

```java
    public void addFlush() {
        // 获取未发送的数据节点
        Entry entry = unflushedEntry;
        if (entry != null) {
            // 将未发送的数据节点 unflushedEntry 转为已发送数据的节点 flushedEntry
            if (flushedEntry == null) {
                flushedEntry = entry;
            }
            do {
                flushed ++;
                if (!entry.promise.setUncancellable()) {
                    int pending = entry.cancel();
                    decrementPendingOutboundBytes(pending, false, true);
                }
                // 继续遍历下个 unflushedEntry 节点，设置下个节点为新的 unflushedEntry，即指向
                // 链表中指向第一个没有写到底层系统缓冲区 Socket 中的节点
                entry = entry.next;
            } while (entry != null);

            // All flushed so reset unflushedEntry
            unflushedEntry = null;
        }
    }
```
