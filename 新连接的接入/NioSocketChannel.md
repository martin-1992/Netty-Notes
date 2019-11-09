### NioSocketChannel
　　为客户端的 Channel，基本流程跟服务端的 Channel 一致。

```java
    public NioSocketChannel(Channel parent, SocketChannel socket) {
        // 调用父类函数
        super(parent, socket);
        // 创建一个 config，配置此 Channel 禁止 Nagle 算法
        config = new NioSocketChannelConfig(this, socket.socket());
    }
```

### AbstractNioByteChannel
　　SelectionKey.OP_READ 为客户端 Channel 的 read 事件，后续将此 Channel 绑定到 selector 上时，表示对读事件比较感兴趣，后续有事件读写则请告诉此 Channel。
  

```java
    protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
        // SelectionKey.OP_READ 为客户端 Channel 的 read 事件，后续将此 Channel 绑定到
        // selector 上时，表示对读事件比较感兴趣，后续有事件读写则请告诉此 Channel
        super(parent, ch, SelectionKey.OP_READ);
    }
```

### AbstractNioChannel 的构造函数<a id='AbstractNioChannel'></a>

- 使用变量 ch 保存创建好的 JDK 底层 Channel；
- 设置此 Channel 对连接的接入事件感兴趣；
- 设置此 Channel 为非阻塞模式。

```java
    protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
        // 创建和此 Channel 相关的组件，包括 Channel 的唯一标识符 id、用于读写的 unsafe、Pipeline
        super(parent);
        // 保存创建好的 JDK 底层 Channel
        this.ch = ch;
        // 感兴趣的 IO 事件，NioServerSocketChannel 传入的是连接的接入事件
        this.readInterestOp = readInterestOp;
        try {
            // 设置此 Channel 为非阻塞模式
            ch.configureBlocking(false);
        } catch (IOException e) {
            try {
                // 出现异常则关闭
                ch.close();
            } catch (IOException e2) {
                if (logger.isWarnEnabled()) {
                    logger.warn(
                            "Failed to close a partially initialized socket.", e2);
                }
            }

            throw new ChannelException("Failed to enter non-blocking mode.", e);
        }
    }
```


### AbstractChannel 的构造函数
　　创建 id（Channel 的唯一标识）、unsafe（底层数据的读写）、Pipeline（业务逻辑的载体，负责该 Channel 数据处理的业务逻辑链）。

```java
    protected AbstractChannel(Channel parent) {
        this.parent = parent;
        id = newId();
        unsafe = newUnsafe();
        pipeline = newChannelPipeline();
    }
```

#### NioServerSocketChannel#javaChannel
　　获取服务端的 Channel，调用父类 AbstractNioChannel#javaChannel 获取。

```java
    @Override
    protected ServerSocketChannel javaChannel() {
        return (ServerSocketChannel) super.javaChannel();
    }
```


#### AbstractNioChannel#javaChannel
　　返回 Channel，变量 ch 在 前面的 [AbstractNioChannel 的构造函数](#AbstractNioChannel) 中提到，保存的是 NioServerSocketChannel。

```java
    private final SelectableChannel ch;

    protected SelectableChannel javaChannel() {
        return ch;
    }
```


### NioSocketChannelConfig 的构造函数
　　调用父类 DefaultSocketChannelConfig 的构造函数，禁止 Nagle 算法。

```java
    private final class NioSocketChannelConfig extends DefaultSocketChannelConfig {
        private volatile int maxBytesPerGatheringWrite = Integer.MAX_VALUE;
        private NioSocketChannelConfig(NioSocketChannel channel, Socket javaSocket) {
            super(channel, javaSocket);
            calculateMaxBytesPerGatheringWrite();
        }
        // ...
```


### DefaultSocketChannelConfig 的构造函数
　　设置客户端的 Channel 禁止 Nagle 算法。Nagle 算法，是将小的数据包集合成大的数据包发送出去。而 Netty 默认情况下，为了使数据能及时发送出去，禁止 Nagle 算法。

```java
    public DefaultSocketChannelConfig(SocketChannel channel, Socket javaSocket) {
        super(channel);
        if (javaSocket == null) {
            throw new NullPointerException("javaSocket");
        }
        this.javaSocket = javaSocket;

        // 默认为 true，即会禁止 Nagle 算法
        if (PlatformDependent.canEnableTcpNoDelayByDefault()) {
            try {
                setTcpNoDelay(true);
            } catch (Exception e) {
                // Ignore.
            }
        }
    }
```

