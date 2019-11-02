### doBind

- [initAndRegister](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/initAndRegister.md)，创建 Channel、初始化配置 Channel、将 Channel 注册到 EventLoop（事件轮询器 Selector）。；
- [doBind0]()，调用 JDK 底层 API 进行端口绑定。

```java
    private ChannelFuture doBind(final SocketAddress localAddress) {
        // 创建、初始化、注册 Channel
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }

        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            // 绑定端口
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // ...
                }
            });
            return promise;
        }
    }
```
