### NioServerSocketChannel的构造函数
　　设置服务端 Channel 的属性和配置。示例代码在 [Netty 的启动过程#initAndRegister](https://github.com/martin-1992/Netty-Notes/edit/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/initAndRegister.md) 讲到通过反射方法创建 ServerSocketChannel，该类会调用父类构造函数，创建 Pipeline。

```java
// 示例代码部分
ServerBootstrap b = new ServerBootstrap();
// 通过 group 将两大线程配置进来
b.group(bossGroup, workerGroup)
        // 设置服务端的 ServerSocketChannel
        .channel(NioServerSocketChannel.class)
```

- AbstractNioByteChannel，**使用变量 ch 保存创建好的 JDK 底层 Channel，设置此 Channel 为对连接的接入事件感兴趣；**
- 设置此 Channel 为非阻塞模式；
- 创建服务端 Channel 相关的组件，**包括 id、unsafe、Pipeline；**
- 创建一个 config，配置服务端的 Channel 禁止 Nagle 算法，小的数据包会尽可能发送出去，降低延时。

### NioServerSocketChannel 的构造函数
　　设置服务端 Channel 的属性和配置，设置该客户端 Channel 禁止 Nagle 算法，SelectionKey.OP_ACCEPT 表示服务端 Channel 对连接的接入事件感兴趣。

```java
    public NioServerSocketChannel(ServerSocketChannel channel) {
        // 调用父类，为 AbstractNioMessageChannel，最终调用 AbstractNioChannel
        super(null, channel, SelectionKey.OP_ACCEPT);
        // 创建一个 config，配置此 Channel 禁止 Nagle 算法
        config = new NioServerSocketChannelConfig(this, javaChannel().socket());
    }
    
    /**
     * AbstractNioMessageChannel 的构造函数
     */
    protected AbstractNioMessageChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
        super(parent, ch, readInterestOp);
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

