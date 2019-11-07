### NioSocketChannel 的构造函数
　　设置该客户端 Channel 的属性和配置。

传入服务端的 Channel 和客户端的 Channel。

继续调用父类函数 super(parent, socket)，对服务端的 Channel 进行配置；
创建客户端 Channel 相关的组件，有 id、unsafe、channelPipeline；
保存创建好的 JDK 底层 Channel，设置此 Channel 为对读事件感兴趣；
设置此 Channel 为非阻塞模式。
创建一个 config，这里的客户端 Channel 通过 new 创建（服务端 Channel 是通过反射方式创建的），配置客户端的 Channel 禁止 Nagle 算法，小的数据包会尽可能发送出去，降低延时。
    public NioSocketChannel(Channel parent, SocketChannel socket) {
        // 调用父类函数，parent 为服务端 Channel，socket 为客户端的 JDK 底层 Channel
        super(parent, socket);
        // 创建一个 config，配置此 Channel 禁止 Nagle 算法
        config = new NioSocketChannelConfig(this, socket.socket());
    }

### NioSocketChannel 的构造函数
　　设置该客户端 Channel 的属性和配置，设置该客户端 Channel 禁止 Nagle 算法。

```java
    public NioSocketChannel(Channel parent, SocketChannel socket) {
        // 调用父类 AbstractNioByteChannel 的构造函数
        super(parent, socket);
        // 创建一个 config，配置此 Channel 禁止 Nagle 算法
        config = new NioSocketChannelConfig(this, socket.socket());
    }
```


### AbstractNioByteChannel 的构造函数
　　SelectionKey.OP_READ 表示客户端 Channel 对读事件感兴趣，后面会将该 Channel 注册到 worker 的 Selector 上，由 worker 的线程组 EventLoopGroup 来处理。

```java
    protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
        super(parent, ch, SelectionKey.OP_READ);
    }
```


### AbstractNioByteChannel 的构造函数

- 保存创建好的 JDK 底层 Channel；
- 设置此 Channel 对读事件感兴趣；
- 设置此 Channel 为非阻塞模式。

```java
    protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
        // 创建和此 Channel 相关的组件，包括 Channel的唯一标识符 id、用于读写的 unsafe、Pipeline
        super(parent);
        // 保存创建好的 JDK 底层 Channel
        this.ch = ch;
        // 感兴趣的读事件
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

- 保存创建此客户端 Channel 的服务端 Channel，也就是在服务端启动过程中，通过反射创建的 Channel；
- 创建 id（Channel 的唯一标识）、unsafe（底层数据的读写）、pipeline（业务逻辑的载体，负责该 Channel 数据处理的业务逻辑链）。

```java
    protected AbstractChannel(Channel parent) {
        // 保存创建此客户端 Channel 的服务端 Channel，即在服务端启动过程中，通过反射创建的 Channel
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
　　使用变量 ch 保存服务端 Channel，在 []() 进行注册。

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

