### ReflectiveChannelFactory#newInstance
　　反射实例化 constructor，而 constructor 是在该类 ReflectiveChannelFactory 的构造函数中传入。

```java
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {

    private final Constructor<? extends T> constructor;
    
    /**
     * 使用 constructor 记录传入的 clazz 类
     */
    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        ObjectUtil.checkNotNull(clazz, "clazz");
        try {
            this.constructor = clazz.getConstructor();
        } catch (NoSuchMethodException e) {
            throw new IllegalArgumentException("Class " + StringUtil.simpleClassName(clazz) +
                    " does not have a public non-arg constructor", e);
        }
    }

    @Override
    public T newChannel() {
        try {
            return constructor.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + constructor.getDeclaringClass(), t);
        }
    }
}
```

　　调用 ReflectiveChannelFactory 的方法的则是示例代码中的 .channel(NioServerSocketChannel.class)。

```java
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.option(ChannelOption.SO_BACKLOG, 1024);
            b.group(bossGroup, workerGroup)
            // 传入 Channel，进行反射实例化
             .channel(NioServerSocketChannel.class)

```


### AbstractBootstrap#channel
　　从该 channel 方法进入可看到将 NioServerSocketChannel.class 传入到 ReflectiveChannelFactory 的构造函数中，然后通过 bind() 方法，最后会调用到 channelFactory.newChannel() 方法进行实例化。

```java
    public B channel(Class<? extends C> channelClass) {
        if (channelClass == null) {
            throw new NullPointerException("channelClass");
        }
        return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
    }
```

