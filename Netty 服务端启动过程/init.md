### init(channel)
　　初始化 Channel，通过引导类 ServerBootstrap 进行配置。<br />
　　这些配置项属性都存在于 AbstractBootstrap 和 ServerBootstrap，一些客户端引导类和服务端引导类公有的属性，会放在 AbstractBootstrap，比如 option。部分只有服务端引导类才有的属性会放到 ServerBootstrap 中，比如 ChannelHandler、childOptions 等。然后调用 init 方法，将引导类的配置赋值给这些变量属性。

```java
    ServerBootstrap b = new ServerBootstrap();
    b.option(ChannelOption.SO_BACKLOG, 1024);
    b.group(bossGroup, workerGroup)
     .channel(NioServerSocketChannel.class)
     .handler(new LoggingHandler(LogLevel.INFO))
     .childHandler(new HttpHelloWorldServerInitializer(sslCtx));
```

### option
　　以 b.option(ChannelOption.SO_BACKLOG, 1024); 为例，使用哈希表来保存配置项的，每次对它操作都调用同步方法锁住该对象（使用 final），保证线程安全性。

```java
public abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> implements Cloneable {

    // 哈希表保存配置项
    private final Map<ChannelOption<?>, Object> options = new LinkedHashMap<ChannelOption<?>, Object>();
    
    public <T> B option(ChannelOption<T> option, T value) {
        if (option == null) {
            throw new NullPointerException("option");
        }
        if (value == null) {
            synchronized (options) {
                options.remove(option);
            }
        } else {
            synchronized (options) {
                options.put(option, value);
            }
        }
        return self();
    }
```

### ServerBootstrap
　　其他属性方法也是同理，有些属性配置是只有服务端才有，所以是继承了 AbstractBootstrap 的服务引导类 ServerBootstrap 才有，比如 ChannelHandler、childOptions 这些。

```java
public class ServerBootstrap extends AbstractBootstrap<ServerBootstrap, ServerChannel> {

    private static final InternalLogger logger = InternalLoggerFactory.getInstance(ServerBootstrap.class);

    private final Map<ChannelOption<?>, Object> childOptions = new LinkedHashMap<ChannelOption<?>, Object>();
    private final Map<AttributeKey<?>, Object> childAttrs = new LinkedHashMap<AttributeKey<?>, Object>();
    private final ServerBootstrapConfig config = new ServerBootstrapConfig(this);
    private volatile EventLoopGroup childGroup;
    private volatile ChannelHandler childHandler;
    
    // ...
}
```

### init
　　对每一个配置项，使用同步方法 synchronized 进行配置，初始化 Channel，保证线程安全。

- 配置 options；
- 获取 Pipeline，在引导类中 .channel(NioServerSocketChannel.class) 中创建；
- 设置 currentChildOptions 和 currentChildAttrs；
- 为 pipeline 添加 ChannelHandler；
- 调用 [SingleThreadEventExecutor#execute](https://github.com/martin-1992/Netty-Notes/blob/56ca8cb154c280855b3caf6746df71e77e7f65f8/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%90%AF%E5%8A%A8/SingleThreadEventExecutor%23Execute.md) 将该非 IO 任务添加到任务队列执行，为 pipeline 添加 [ServerBootstrapAcceptor]()，用于接收客户端连接 Socket 创建连接后，对连接的初始化。

```java
    @Override
    void init(Channel channel) throws Exception {
        final Map<ChannelOption<?>, Object> options = options0();
        // 在初始化 Channel，分别使用两个 synchronized 来上锁，代替使用对方法进行
        // synchronized。减少锁的粒度，从方法级别到对象级别
        synchronized (options) {
            setChannelOptions(channel, options, logger);
        }

        final Map<AttributeKey<?>, Object> attrs = attrs0();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
                @SuppressWarnings("unchecked")
                AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
                channel.attr(key).set(e.getValue());
            }
        }
        // 获取 Pipeline
        ChannelPipeline p = channel.pipeline();

        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions;
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(newOptionArray(0));
        }
        synchronized (childAttrs) {
            currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));
        }

        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) throws Exception {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                // 添加 ChannelHandler
                if (handler != null) {
                    pipeline.addLast(handler);
                }
                // ServerBootstrapAcceptor，负责接收客户端连接 Socket 创建连接后，对连接的初始化
                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }
```

