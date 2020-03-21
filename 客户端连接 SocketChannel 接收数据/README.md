### NioEventLoop#processSelectedKey
　　检测到 read 操作或 accept 操作，则调用 read() 方法。该方法有两个实现类。

- NioSocketChannel.read() 是读数据，对应 SelectionKey.OP_READ；
- NIOServerSocketChannel.read() 是创建客户端连接，对应 SelectionKey.OP_ACCEPT。

```java
    if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
        // AbstractNioMessageChannel 的 read 方法，进到 AbstractNioMessageChannel
        unsafe.read();
    }
```

### NioSocketChannel#read

- 获取配置中的 ByteBuf 的分配器；
- 创建自适应数据大小的分配器并初始化，根据实际接收数据的情况，决定接下来是否要扩容或缩容 ByteBuf；
- doReadBytes(byteBuf)，实际读取数据的逻辑，将其放入 byteBuf 中；
- [AdaptiveRecvByteBufAllocator#lastBytesRead]() 会调用 record() 方法，根据这次接收数据大小，来调整接下来的 ByteBuf 大小；
    1. 缩容需要两次判断，同时缩容是容量表的索引减 1；
    2. 扩容只要一次判断，而且扩容的步伐比较大，容量表的索引加 4；
    3. 会限制一个 SocketChannel 读取数据的次数，默认为 16 次。因为一个 NioEventLoop 上绑定多个 Channel，防止一个 Channel 不断读取数据，导致其他 Channel 饿死，读取不到数据。
- [pipeline.fireChannelRead(byteBuf)]()，pipeline 上执行，业务逻辑的处理在这个地方，将读到的数据进行传递，同样是从 headContext 开始往下传播；
- 当读取完，会记录这次读事件总共读了多少数据，调用 record 计算下次分配的 ByteBuf 大小；
    1. 前面的 AdaptiveRecvByteBufAllocator#lastBytesRead 调整 ByteBuf 大小，是存在多次读取，因为一次读取不完，于是进行多次读取，然后调整 ByteBuf 大小。即一次读事件，多次读取；
    2. 这里是对总的读取大小，进行调整，避免下次需要分多次才能读取完一个读事件的数据。同时在调整中，扩容是一次判断，且扩容较大。
- 读取完，调用 [pipeline.fireChannelReadComplete()]() 进行传播。
    1. pipeline.fireChannelReadComplete()，为一次读事件的完成，进行 pipeline 的传播；
    2. pipeline.fireChannelRead(byteBuf)，为一次读数据的完成，一次读事件处理可能会包含多次读数据操作。

```java
        @Override
        public final void read() {
            // 在 NioSocketChannel 的构造函数，会进行配置，主要是配置此 Channel 禁止 Nagle 算法
            final ChannelConfig config = config();
            // 输入是否关闭，是则移除读事件，并返回
            if (shouldBreakReadReady(config)) {
                clearReadPending();
                return;
            }
            // 获取该 Channel 的 pipeline
            final ChannelPipeline pipeline = pipeline();
            // 获取配置中的 ByteBuf 的分配器
            final ByteBufAllocator allocator = config.getAllocator();
            // 创建自适应数据大小的分配器，根据实际接收数据的情况，决定接下来是否要扩容或缩容 ByteBuf
            final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
            // 重新载入配置，初始化
            allocHandle.reset(config);

            ByteBuf byteBuf = null;
            boolean close = false;
            try {
                do {
                    // 判断下次接收数据的 ByteBuf 大小，这里会判断是使用直接内存 directBuffer，还是堆内存 heapBuffer
                    byteBuf = allocHandle.allocate(allocator);
                    // 读取数据，将其放入 byteBuf 中
                    // 根据这次接收数据大小，来调整接下来的 ByteBuf 大小
                    allocHandle.lastBytesRead(doReadBytes(byteBuf));
                    // allocHandle.lastBytesRead(doReadBytes(byteBuf) 会记录读取数据的大小，保存到变量 lastBytesRead 中，
                    // 这里判断是否小于等于 0，是表示这次没有接收到数据，则释放 ByteBuf，并返回
                    if (allocHandle.lastBytesRead() <= 0) {
                        // nothing was read. release the buffer.
                        byteBuf.release();
                        byteBuf = null;
                        close = allocHandle.lastBytesRead() < 0;
                        if (close) {
                            // There is nothing left to read as we received an EOF.
                            readPending = false;
                        }
                        break;
                    }

                    // 记录读取的次数 + 1
                    allocHandle.incMessagesRead(1);
                    readPending = false;
                    // pipeline 上执行，业务逻辑的处理在这个地方，将读到的数据进行传递，同样
                    // 是从 headContext 开始往下传播
                    pipeline.fireChannelRead(byteBuf);
                    byteBuf = null;
                    // 读取一次，就跳出循环
                } while (allocHandle.continueReading());

                // 记录这次读事件总共读了多少数据，调用 record 计算下次分配的 ByteBuf 大小
                allocHandle.readComplete();
                // 读取完，进行传播
                pipeline.fireChannelReadComplete();

                if (close) {
                    closeOnRead(pipeline);
                }
            } catch (Throwable t) {
                handleReadException(pipeline, byteBuf, t, close, allocHandle);
            } finally {
                if (!readPending && !config.isAutoRead()) {
                    removeReadOp();
                }
            }
        }
    }
```


#### DefaultChannelConfig#getAllocator
　　获取 ByteBuf 的分配器。

```java
    @Override
    public ByteBufAllocator getAllocator() {
        return allocator;
    }
```

#### AbstractChannel#recvBufAllocHandle
　　实例化一个 [AdaptiveRecvByteBufAllocator#Handle]()，用于调整 ByteBuf 接收数据的大小。

```java
    @Override
    public RecvByteBufAllocator.Handle recvBufAllocHandle() {
        if (recvHandle == null) {
            recvHandle = config().getRecvByteBufAllocator().newHandle();
        }
        return recvHandle;
    }
```

#### DefaultMaxMessagesRecvByteBufAllocator#allocate

- 调用 [AdaptiveRecvByteBufAllocator#HandleImpl#guess]() 来获取下次接收数据的 ByteBuf 大小，该方法通过 record() 进行扩容或缩容；
- AbstractByteBufAllocator#ioBuffer，判断返回的是直接内存 directBuffer，还是堆内存 heapBuffer。

```java
    @Override
    public ByteBuf allocate(ByteBufAllocator alloc) {
        return alloc.ioBuffer(guess());
    }

    @Override
    public ByteBuf ioBuffer(int initialCapacity) {
        if (PlatformDependent.hasUnsafe()) {
            return directBuffer(initialCapacity);
        }
        return heapBuffer(initialCapacity);
    }
```

#### NioSocketChannel#doReadBytes
　　读取数据，放入 ByteBuf 中。

```java
    @Override
    protected int doReadBytes(ByteBuf byteBuf) throws Exception {
        final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
        allocHandle.attemptedBytesRead(byteBuf.writableBytes());
        return byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead());
    }
```

#### DefaultMaxMessagesRecvByteBufAllocator#continueReading
　　判断是否要继续读取连接或数据。

- config.isAutoRead()，默认为 1；
- respectMaybeMoreData，默认为 true。这里 !respectMaybeMoreData 表示 false，表示不会获取更多数据，即读取一次就会打破循环。如果为 true，最大会读取 16 次；
- 在读取数据时，totalBytesRead 不为 0；
- maxMessagePerRead，默认为 16，即读取数据次数，最大为 16 次；


```java
    @Override
    public boolean continueReading() {
        return continueReading(defaultMaybeMoreSupplier);
    }

    @Override
    public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {
        return config.isAutoRead() &&
               (!respectMaybeMoreData || maybeMoreDataSupplier.get()) &&
               totalMessages < maxMessagePerRead &&
               totalBytesRead > 0;
    }
```