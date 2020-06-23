### 获取 NioEventLoop
　　在 [MultithreadEventLoopGroup#register]() 方法中，会从 EventLoopGroup 中选择其中一个 EventLoop 进行绑定，选择方法有两种，分别为 [PowerOfTwoEventExecutorChooser#next() 和 GenericEventExecutorChooser#next()](https://github.com/martin-1992/Netty-Notes/blob/932d84af92758d157f544186146f943a6c2a5778/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%88%9B%E5%BB%BA/newChooser.md)。

```java
    @Override
    public ChannelFuture register(Channel channel) {
        // next() 方法返回一个 NioEventLoop
        return next().register(channel);
    }
```

　　**当 EventLoopGroup 的大小为 2 的幂时，则会使用位运算，等价于求余数 %，提高性能。** 而 Netty 的 EventLoopGroup，比如 NioEventLoopGroup 的参数默认是一个核心 2 个线程的。

```java
    @Override
    public EventExecutor next() {
        // 使用位运算，通过 & 方式返回一个 EventLoop
        return executors[idx.getAndIncrement() & executors.length - 1];
    }
```

### 获取时间轮的下个格子
　　同上面一样，在 [Worker#run](https://github.com/martin-1992/Netty-Notes/blob/master/%E6%97%B6%E9%97%B4%E8%BD%AE%20HashedWheelTimer/Worker.md) 中，时间轮的格子数为 2 的幂。<br />
　　输的格子数，Netty 会自动纠正为 2 的幂。**Netty 通过对格子数不断加一来获取下个格子，加一的格子 & 格子数，得到类似 % 的效果，获取正确的格子索引。**

```java
    private final class Worker implements Runnable {
        // 待处理（执行）的定时任务集合
        private final Set<Timeout> unprocessedTimeouts = new HashSet<Timeout>();

        // 格子计数，加一即移到下个格子
        private long tick;

        @Override
        public void run() {
            // ...
            do {
                // 需要 sleep 到下一个格子检查定时任务的时间
                final long deadline = waitForNextTick();
                if (deadline > 0) {
                    // 获取格子索引，因为 tick 是一直往前推移（加一的），mask 是 2 的次方值，
                    // 使用位运算 & 来获取索引值
                    int idx = (int) (tick & mask);
                    // ...
                }
            } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);
            // ...
```

### 优化原生的 Selector
　　在 [NioEventLoop 的构造函数](https://github.com/martin-1992/Netty-Notes/blob/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%88%9B%E5%BB%BA/NioEventLoop%20%E7%9A%84%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0.md) 中，会调用 openSelector 方法创建一个 Selector（IO 事件轮询器）。<br />
　　这里对其进行优化，原生的 Selector 中 selectedKeys 和 publicSelectedKeys 都使用 hashSet 实现，**Netty 替换为用数组实现，便于 Netty 遍历 SelectionKey，效率更高。**

```java
    SelectedSelectionKeySet() {
        keys = new SelectionKey[1024];
    }
```

　　Netty 使用优化后的 keySet，进行数组遍历。

```java
    // NioEventLoop#processSelectedKeysOptimized
    private void processSelectedKeysOptimized() {
        // 遍历 keySet，获取每个 key 对应的 Channel，Netty 使用数组形式的 keySet，遍历更高效
        for (int i = 0; i < selectedKeys.size; ++i) {
            // 这里 key 是在 [Netty 服务端启动过程#AbstractNioChannel#register] 进行注册的，
            // 每个 key 的 attachment 对应一个 AbstractNioChannel
            final SelectionKey k = selectedKeys.keys[i];
            // ...
```

### 不同的 SelectionKey 判断
　　服务端 ServerSocketChannel 绑定的 NioEventLoop 会死循环调用 run() 方法，该方法会获取 SelectionKey。然后调用 [processSelectedKey()](https://github.com/martin-1992/Netty-Notes/blob/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%90%AF%E5%8A%A8/processSelectedKeys.md) 进行处理，包含几种对 SelectionKey 的处理。<br />
　　readyOps 为 k.readyOps()，即 Channel 的 SelectionKey。这里对 SelectionKey 的判断，是通过使用位运算相与 &。如下 readyOps 为 SelectionKey.OP_READ，则相与 & 不为 0，于是判断为 SocketChannel 进行读事件的操作。

```java
    if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
        unsafe.read();
    }
```

### 二分法获取 ByteBuf 的大小
　　[AdaptiveRecvByteBufAllocator](https://github.com/martin-1992/Netty-Notes/blob/master/%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%BF%9E%E6%8E%A5%20SocketChannel%20%E6%8E%A5%E6%94%B6%E6%95%B0%E6%8D%AE/AdaptiveRecvByteBufAllocator.md) 为自适应数据大小的分配器，根据这次接收数据的实际大小，调整接下来的 ByteBuf 大小。
　　Netty 会使用一个容量表，0~496 为 16 递增，而 512~1073741824 为两倍递增，即容量表为 [16, 32, 48, ... , 496, 512, 1024, 2048, ... , 1073741824]。**通过二分法，从容量表中找到 ByteBuf 最适合的大小。**

```java
    private static int getSizeTableIndex(final int size) {
        for (int low = 0, high = SIZE_TABLE.length - 1;;) {
            if (high < low) {
                return low;
            }
            if (high == low) {
                return high;
            }

            // 二分法
            int mid = low + high >>> 1;
            // 左中位数
            int a = SIZE_TABLE[mid];
            // 右中位数
            int b = SIZE_TABLE[mid + 1];
            // 二分法判断
            if (size > b) {
                low = mid + 1;
            } else if (size < a) {
                high = mid - 1;
            } else if (size == a) {
                return mid;
            } else {
                return mid + 1;
            }
        }
    }
```

### 不断调整 ByteBuf 大小
　　在[接收数据的 ByteBuf](https://github.com/martin-1992/Netty-Notes/tree/master/%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%BF%9E%E6%8E%A5%20SocketChannel%20%E6%8E%A5%E6%94%B6%E6%95%B0%E6%8D%AE) 中，一次读事件可能包含多次读操作，如果一次读取不完的话，会进行扩容，容量表索引加 4。<br />
　　比如容量表为 [16, 32, 48, 64, 80, ...]，开始时 ByteBuf 大小为 16，这时没读取完数据，进行第二次读取，调整大小，容量表索引加 4，所以调整后的 ByteBuf 大小为 80，每次读取都会调整 ByteBuf 大小。<br />
　　当一次读事件完成后，会记录这次读事件总共读了多少数据，调用 record 计算下次分配的 ByteBuf 大小。

- allocHandle.lastBytesRead(doReadBytes(byteBuf))，一次读取不完，进行多次读取。然后调整 ByteBuf 大小。即一次读事件，多次读取操作；
- allocHandle.readComplete()，读事件完成。记录这次读事件总共读了多少数据，调用 record 计算下次分配的 ByteBuf 大小。

```java
@Override
    public final void read() {
        // ...
        try {
            do {
                // 判断下次接收数据的 ByteBuf 大小，这里会判断是使用直接内存 directBuffer，还是堆内存 heapBuffer
                byteBuf = allocHandle.allocate(allocator);
                // 读取数据，将其放入 byteBuf 中
                // 根据这次接收数据大小，来调整接下来的 ByteBuf 大小
                allocHandle.lastBytesRead(doReadBytes(byteBuf));
                // ...
            } while (allocHandle.continueReading());

            // 记录这次读事件总共读了多少数据，调用 record 计算下次分配的 ByteBuf 大小
            allocHandle.readComplete();
            // ...
```
