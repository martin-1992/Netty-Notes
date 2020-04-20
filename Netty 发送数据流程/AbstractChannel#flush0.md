### AbstractChannel#flush0
　　核心方法为 doWrite，将消息发送出去。

```java
    @SuppressWarnings("deprecation")
    protected void flush0() {
        if (inFlush0) {
            // Avoid re-entrance
            return;
        }

        final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
        if (outboundBuffer == null || outboundBuffer.isEmpty()) {
            return;
        }
        
        // ... 
        
        try {
            // 核心逻辑
            doWrite(outboundBuffer);
        } catch (Throwable t) {
            // ..
            }
        } finally {
            inFlush0 = false;
        }
    }
```

### NioSocketChannel#doWrite
　　消息发送的流程。

- 将要发送的数据节点 flushedEntry 包装成 ByteBuffer，添加到 ByteBuffer 的数组 nioBuffers；
- 调用底层方法 ch.write(nioBuffers)，将数据写入，分为单次写入 nioBuffers[0] 和批量写入 nioBuffers；
- 根据 buffer 剩余的大小和这次写入的消息数据大小，动态调整 buffer 大小，判断是否要扩容或缩容；
- 从 ChannelOutboundBuffer 中移除已经发送出去的数据；
- 调用 incompleteWrite 方法。
    1. 如果消息数据都已写入，则注册一个 OP_WRITE 写事件，表示可以发送。然后在 NioEventLoop#processSelectedKey 会检查到 OP_WRITE，调用底层的 ch.unsafe().forceFlush() 发送消息数据；
    2. 如果循环 16 次后消息都还没写完，则创建一个 task 来执行。让其他线程可以继续执行任务，不会阻塞太长时间;
    3. 这里设置为 16 次，是防止 NioEventLoop 一直在处理该 Channel 的消息发送，导致其它 Channel 长时间等待，类似线程饿死的情况。

```java
    @Override
    protected void doWrite(ChannelOutboundBuffer in) throws Exception {
        SocketChannel ch = javaChannel();
        // 写数据，最多写 16 次，writeSpinCount 默认为 16
        int writeSpinCount = config().getWriteSpinCount();
        do {
            // 如果传入的 ChannelOutboundBuffer 为空，表示没有数据可发送，直接返回
            if (in.isEmpty()) {
                // 数据都写完了，清除 OP_WRITE 标记
                clearOpWrite();
                return;
            }

            // 默认发送的最大数据大小为 Integer.MAX_VALUE，尽量多写数据
            int maxBytesPerGatheringWrite = ((NioSocketChannelConfig) config).getMaxBytesPerGatheringWrite();
            // 将要发送的数据节点 flushedEntry 包装成 ByteBuffer，添加到 ByteBuffer 的数组，该 ByteBuffer
            // 的数组大小最大为 1024，即包含 1024 个 flushedEntry
            ByteBuffer[] nioBuffers = in.nioBuffers(1024, maxBytesPerGatheringWrite);
            int nioBufferCnt = in.nioBufferCount();

            // 进行发送数据，nioBufferCnt 为 1，表示只有一条数据要发送。nioBufferCnt 大于 1，
            // 则调用 default 方法批量数据
            switch (nioBufferCnt) {
                case 0:
                    // We have something else beside ByteBuffers to write so fallback to normal writes.
                    writeSpinCount -= doWrite0(in);
                    break;
                case 1: {
                    // 只有一条数据要发送
                    ByteBuffer buffer = nioBuffers[0];
                    // 写完一次数据后，获取 buffer 的剩余的大小
                    int attemptedBytes = buffer.remaining();
                    // 单条数据，调用底层的 Channel.write 方法
                    final int localWrittenBytes = ch.write(buffer);
                    // 当（发送）写不出去事件时，会注册一个 OP_WRITE 事件，等能写进去时，在通知来写
                    if (localWrittenBytes <= 0) {
                        incompleteWrite(true);
                        return;
                    }
                    // 如果剩余大小 attemptedBytes 和当前要发送的数据 localWrittenBytes 相等，则进行扩容
                    adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite);
                    // 从 ChannelOutboundBuffer 中移除已经发送出去的数据
                    in.removeBytes(localWrittenBytes);
                    // 减少写的次数，writeSpinCount 最大为 16 次，即最多尝试写 16 次
                    --writeSpinCount;
                    break;
                }
                default: {
                    long attemptedBytes = in.nioBufferSize();
                    // 批量数据调用的
                    final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                    if (localWrittenBytes <= 0) {
                        // 缓存区满了，写不进去，注册写事件，在 NioEventLoop#processSelectedKey 会检查到 OP_WRITE，
                        // 先发送数据
                        incompleteWrite(true);
                        return;
                    }
                    // Casting to int is safe because we limit the total amount of data in the nioBuffers to int above.
                    adjustMaxBytesPerGatheringWrite((int) attemptedBytes, (int) localWrittenBytes,
                            maxBytesPerGatheringWrite);
                    in.removeBytes(localWrittenBytes);
                    --writeSpinCount;
                    break;
                }
            }
        } while (writeSpinCount > 0);
        // 当 16 次用完后，则注册一个 OP_WRITE 事件，没用完则创建一个 task 来执行，
        // 让其他线程可以执行任务，不会阻塞太长时间
        incompleteWrite(writeSpinCount < 0);
    }
```

#### AbstractNioByteChannel#incompleteWrite
　　判断是注册 OP_WRITE 事件还是创建任务继续执行未发送完的消息。

```java
    protected final void incompleteWrite(boolean setOpWrite) {
        // writeSpinCount < 0 为 true 的情况，
        if (setOpWrite) {
            // 注册一个 OP_WRITE 事件
            setOpWrite();
        } else {
            // writeSpinCount < 0 为 false 的情况，即还有数据没写完，则创建一个 task 来执行，
            // 让其他线程执行任务，不会阻塞太长时间
            clearOpWrite();
            eventLoop().execute(flushTask);
        }
    }
    
    protected final void setOpWrite() {
        final SelectionKey key = selectionKey();
        if (!key.isValid()) {
            return;
        }
        final int interestOps = key.interestOps();
        if ((interestOps & SelectionKey.OP_WRITE) == 0) {
            // 对 key 注册一个 OP_WRITE 事件
            key.interestOps(interestOps | SelectionKey.OP_WRITE);
        }
    }
```

#### NioEventLoop#processSelectedKey
　　调用方法 setOpWrite() 方法注册一个 OP_WRITE 事件后，在 NioEventLoop 中会轮询检查到 OP_WRITE 事件，然后调用底层 channel 执行 flush 发送消息的操作。

```java
    private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
            // ...
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
                ch.unsafe().forceFlush();
            }
```


