### pipeline 的初始化
　　示例代码中，实例化 [NioServerSocketChannel](https://github.com/martin-1992/Netty-Notes/blob/2571fcbffe2cb9588dddf59e76c9b885a0bf8458/%E6%96%B0%E8%BF%9E%E6%8E%A5%E7%9A%84%E6%8E%A5%E5%85%A5/NioServerSocketChannel.md) 时会创建 pipeline。

```java
// 示例代码部分
ServerBootstrap b = new ServerBootstrap();
// 通过 group 将两大线程配置进来
b.group(bossGroup, workerGroup)
        // 设置服务端的 ServerSocketChannel
        .channel(NioServerSocketChannel.class)
```

### AbstractChannel 的构造函数
　　实例化的 NioServerSocketChannel，最终会调用 AbstractChannel 的构造函数，创建 pipeline。

```java
    protected AbstractChannel(Channel parent) {
        // 保存创建此客户端 Channel 的服务端 Channel，也就是在服务端启动
        // 过程中，通过反射创建的 Channel
        this.parent = parent;
        id = newId();
        unsafe = newUnsafe();
        pipeline = newChannelPipeline();
    }
    
    protected DefaultChannelPipeline newChannelPipeline() {
        return new DefaultChannelPipeline(this);
    }
```

### DefaultChannelPipeline 的构造函数
　　pipeline 默认情况下会创建两个节点 [HeadContext](https://github.com/martin-1992/Netty-Notes/blob/master/pipeline%20%E8%A7%A3%E6%9E%90/HeadContext.md) 和 [TailContext](https://github.com/martin-1992/Netty-Notes/blob/master/pipeline%20%E8%A7%A3%E6%9E%90/TailContext.md)，使用哨兵模式，为双向链表。

```java
    protected DefaultChannelPipeline(Channel channel) {
        // 检查不为空，则将 Channel 进行保存
        this.channel = ObjectUtil.checkNotNull(channel, "channel");
        succeededFuture = new SucceededChannelFuture(channel, null);
        voidPromise =  new VoidChannelPromise(channel, true);

        // 会创建两个节点
        tail = new TailContext(this);
        head = new HeadContext(this);
        // 双向链表
        head.next = tail;
        tail.prev = head;
    }
```
