### AbstractUnsafe#write
　　将消息 message 添加到 outboundBuffer 中。

- 过滤掉非 ByteBuf 对象和非 FileRegion，将非直接内存转换成直接内存 DirectBuffer；
- 计算待发送数据 message 的大小；
- 将消息 message 添加到 buffer 中。

```java
        @Override
        public final void write(Object msg, ChannelPromise promise) {
            assertEventLoop();

            ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            // outboundBuffer 为空，表示该 Channel 已经关闭，则释放消息，防止内存泄漏
            if (outboundBuffer == null) {
                safeSetFailure(promise, newWriteException(initialCloseCause));
                ReferenceCountUtil.release(msg);
                return;
            }

            int size;
            try {
                // 过滤掉非 ByteBuf 对象和非 FileRegion，将非直接内存转换成直接内存 DirectBuffer
                msg = filterOutboundMessage(msg);
                // 计算待发送数据 message 的大小
                size = pipeline.estimatorHandle().size(msg);
                if (size < 0) {
                    size = 0;
                }
            } catch (Throwable t) {
                safeSetFailure(promise, t);
                ReferenceCountUtil.release(msg);
                return;
            }
            // 将消息 message 添加到 buffer 中
            outboundBuffer.addMessage(msg, size, promise);
        }
```        

### AbstractNioByteChannel#filterOutboundMessage
　　过滤掉非 ByteBuf 对象和非 FileRegion，将非直接内存转换成直接内存 DirectBuffer

```java
    @Override
    protected final Object filterOutboundMessage(Object msg) {
        if (msg instanceof ByteBuf) {
            ByteBuf buf = (ByteBuf) msg;
            if (buf.isDirect()) {
                return msg;
            }

            return newDirectBuffer(buf);
        }

        if (msg instanceof FileRegion) {
            return msg;
        }

        throw new UnsupportedOperationException(
                "unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
    }
```          

### DefaultMessageSizeEstimator#HandleImpl#size
　　以 ByteBuf 为例，使用写指针 - 读指针，即可知道要发送数据的大小。

```java
    @Override
    public int size(Object msg) {
        if (msg instanceof ByteBuf) {
            return ((ByteBuf) msg).readableBytes();
        }
        if (msg instanceof ByteBufHolder) {
            return ((ByteBufHolder) msg).content().readableBytes();
        }
        if (msg instanceof FileRegion) {
            return 0;
        }
        return unknownSize;
    }
    
    /**
    * AbstractByteBuf#readableBytes
    */
    @Override
    public int readableBytes() {
        return writerIndex - readerIndex;
    }
```

### ChannelOutboundBuffer#addMessage
　　将消息数据包装成节点 entry，添加链表尾部。Entry 为一个节点，多个 Entry 组成一个单向链表，如下图。

- tailEntry，为缓冲区 ChannelOutboundBuffer 的最后一个节点；
- unFushedEntry，链表中指向第一个没有写到底层系统缓冲区 Socket 中的节点；
- flushedEntry，链表中指向第一个写入到底层系统缓冲区 Socket 中的节点。

![avatar](photo_2.png)

```java
    public void addMessage(Object msg, int size, ChannelPromise promise) {
        // 将新传入的消息 msg 包装为待发送的节点
        Entry entry = Entry.newInstance(msg, size, total(msg), promise);
        if (tailEntry == null) {
            flushedEntry = null;
        } else {
            // 尾节点不为空时，将新传入的消息节点添加到链表尾部，设置为新的尾节点
            Entry tail = tailEntry;
            tail.next = entry;
        }
        tailEntry = entry;
        if (unflushedEntry == null) {
            unflushedEntry = entry;
        }
        // 动态调整 ChannelOutboundBuffer 的大小，即加上当前数据 size 的大小，以便下次能接收更多数据
        incrementPendingOutboundBytes(entry.pendingSize, false);
    }
```

#### ChannelOutboundBuffer#incrementPendingOutboundBytes
　　动态调整 ChannelOutboundBuffer 的大小，即加上当前数据 size 的大小，以便下次能接收更多数据。

```java
    private static final AtomicLongFieldUpdater<ChannelOutboundBuffer> TOTAL_PENDING_SIZE_UPDATER =
            AtomicLongFieldUpdater.newUpdater(ChannelOutboundBuffer.class, "totalPendingSize");

    private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
        if (size == 0) {
            return;
        }

        // size 为当前消息的大小，加上当前消息大小，动态 ChannelOutboundBuffer 的大小，以便下次能接收更多数据
        long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);
        // 如果调整后的 ChannelOutboundBuffer 的大小超过高水位线，则将状态设置为不可写
        if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) {
            setUnwritable(invokeLater);
        }
    }
```