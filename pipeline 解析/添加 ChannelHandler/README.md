### 添加 ChannelHandler
　　以示例代码为例，配置 childHandler 时会传入 ChannelInitializer，然后重写 initChannel 方法，调用 addLast 添加 ChannelHandler。

```java
    ServerBootstrap b = new ServerBootstrap();
    // 通过 group 将两大线程配置进来
    b.group(bossGroup, workerGroup)
            // 设置服务端的 ServerSocketChannel
            .channel(NioServerSocketChannel.class)
            // 给后面指定的每个客户端的连接，设置 TCP 的基本属性
            .childOption(ChannelOption.TCP_NODELAY, true)
            // 每次创建客户端连接时，绑定一些基本的属性
            .childAttr(AttributeKey.newInstance("childAttr"), "childAttrValue")
            // 服务端启动过程中的处理逻辑
            .handler(new ServerHandler())
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) {
                    // 
                    ch.pipeline().addLast(new ChannelInboundHandlerAdapter());
                    ch.pipeline().addLast(new ChannelOutboundHandlerAdapter());
                    //..

                }
            });
```


### DefaultChannelPipeline#addLast
　　添加一个或多个 ChannelHandler，通过遍历方法来添加多个 ChannelHandler。

```java
    @Override
    public final ChannelPipeline addLast(ChannelHandler... handlers) {
        // 这里 executor 为 null
        return addLast(null, handlers);
    }

    @Override
    public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers) {
        if (handlers == null) {
            throw new NullPointerException("handlers");
        }
        // 遍历每个 handler，调用 addLast 重载方法
        for (ChannelHandler h: handlers) {
            if (h == null) {
                break;
            }
            addLast(executor, null, h);
        }

        return this;
    }
```


### DefaultChannelPipeline#addLast
　　添加 ChannelHandler 的核心处理逻辑，流程如下：

- [checkMultiplicity]()，判断 ChannelHandler 是否能重复添加，即一个 ChannelHandler 添加到多个 Channel 里。如果 Handler 不为共享式，且已添加，则抛出异常；
- [newContext](https://github.com/martin-1992/Netty-Notes/blob/master/pipeline%20%E8%A7%A3%E6%9E%90/%E6%B7%BB%E5%8A%A0%20ChannelHandler/newContext.md)，将 ChannelHandler 包装成节点 ChannelHandlerContext（[pipeline 的初始化](https://github.com/martin-1992/Netty-Notes/blob/master/pipeline%20%E8%A7%A3%E6%9E%90/pipeline%20%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96.md) 使用哨兵模式，添加 [HeadContext](https://github.com/martin-1992/Netty-Notes/blob/master/pipeline%20%E8%A7%A3%E6%9E%90/HeadContext.md) 和 [TailContext](https://github.com/martin-1992/Netty-Notes/blob/master/pipeline%20%E8%A7%A3%E6%9E%90/TailContext.md)，使用变量保存 handler、pipeline、executor、inbound、outbound；
- [addLast0](https://github.com/martin-1992/Netty-Notes/blob/master/pipeline%20%E8%A7%A3%E6%9E%90/%E6%B7%BB%E5%8A%A0%20ChannelHandler/addLast0.md)，将包装好的节点 ChannelHandlerContext，插入到链表 pipeline 中。InBound 和 OutBound 事件的传播都是通过遍历链表进行的，InBound 从 head 节点开始往后传播，OutBound 从 tail 节点开始往前传播；
- [callHandlerAdded0](https://github.com/martin-1992/Netty-Notes/blob/master/pipeline%20%E8%A7%A3%E6%9E%90/%E6%B7%BB%E5%8A%A0%20ChannelHandler/callHandlerAdded0.md)，添加成功后，会回调用户重写的 initChannel 方法。

```java
    @Override
    public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized (this) {
            // 判断 ChannelHandler 是否被重复添加
            checkMultiplicity(handler);
            // 创建节点
            newCtx = newContext(group, filterName(name, handler), handler);
            // 添加到链表
            addLast0(newCtx);

            // 如果没注册上，则进行注册
            if (!registered) {
                newCtx.setAddPending();
                callHandlerCallbackLater(newCtx, true);
                return this;
            }
            // 获取新包装节点的线程工厂 executor（每个 NioEventLoop 线程都有一个 executor）
            EventExecutor executor = newCtx.executor();
            // 如果当前线程不为 EventLoop 线程，则创建一个任务（调用 callHandlerAdded0）添加到任务队列中
            if (!executor.inEventLoop()) {
                callHandlerAddedInEventLoop(newCtx, executor);
                return this;
            }
        }
        // 调用回调添加完成事件
        callHandlerAdded0(newCtx);
        return this;
    }
```

