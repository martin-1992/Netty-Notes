### Netty 发送数据的流程
　　如下为 Netty 中的示例代码，在 channelRead 方法中会写数据。当完成后，会调用 channelReadComplete，将数据发送出去。

```java
@Sharable
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    
    /**
     * 写数据
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ctx.write(msg);
    }

    /**
     * 发送数据
     */
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // Close the connection when an exception is raised.
        cause.printStackTrace();
        ctx.close();
    }
}
```

- ctx.write()，ctx 表示 pipeline 上的一个节点 ChannelHandlerContext；
    1. 在 [AbstractChannelHandlerContext#write](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E5%8F%91%E9%80%81%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A8%8B/AbstractChannelHandlerContext%23write.md) 方法中，会找到下个节点 ChannelHandlerContext，调用其 invokeWrite 方法，将消息数据传到下个节点；
    2. 在 [AbstractUnsafe#write](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E5%8F%91%E9%80%81%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A8%8B/AbstractChannel%23write.md) 方法中，调用 outboundBuffer.addMessage()，将数据添加到该 buffer 中。
- ctx.flush()，在 [AbstractChannelHandlerContext#flush](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E5%8F%91%E9%80%81%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A8%8B/AbstractChannelHandlerContext%23flush.md) 方法中，会找到下个节点 ChannelHandlerContext， 调用其 invokeFlush 方法，包含两步；
    1. outboundBuffer.addFlush()，将未发送的数据节点 unflushedEntry 转为已发送数据的节点 flushedEntry；
    2. 在 [AbstractChannel#flush0]() 调用 doWrite 方法，调用底层方法 ch.write() 发送消息；
    3. 注意，当写数据的缓冲区不够时，会注册一个 OP_WRITE 事件（表示可以写数据进去）。写完后要及时关闭。因为在 NioEventLoop#processSelectedKey 中会轮询检查到 OP_WRITE 事件，导致每次循环会检测到 OP_WRITE 事件。
