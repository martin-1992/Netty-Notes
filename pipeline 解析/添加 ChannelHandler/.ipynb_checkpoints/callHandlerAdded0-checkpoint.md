### DefaultChannelPipeline#callHandlerAdded0

- [callHandlerAdded]()，调用 ChannelInitializer 的 handlerAdded 方法，该方法会调用示例代码中重写的 initChannel 方法。即在完成添加后 ChannelHandler 后，回调 initChannel 方法。
- 不成功，则删除包装的节点 ChannelHandlerContext。

```java
    private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
        try {
            ctx.callHandlerAdded();
        } catch (Throwable t) {
            boolean removed = false;
            try {
                remove0(ctx);
                ctx.callHandlerRemoved();
                removed = true;
            } catch (Throwable t2) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Failed to remove a handler: " + ctx.name(), t2);
                }
            }

            if (removed) {
                fireExceptionCaught(new ChannelPipelineException(
                        ctx.handler().getClass().getName() +
                        ".handlerAdded() has thrown an exception; removed.", t));
            } else {
                fireExceptionCaught(new ChannelPipelineException(
                        ctx.handler().getClass().getName() +
                        ".handlerAdded() has thrown an exception; also failed to remove.", t));
            }
        }
    }
```

### AbstractChannelHandlerContext#callHandlerAdded

- 使用自旋 CAS 方式，为添加到链表的节点设置添加成功的属性；
- 调用 ChannelInitializer 的 handlerAdded 方法，该方法会调用示例代码中重写的 initChannel 方法。即在完成添加后 ChannelHandler 后，回调 initChannel 方法；

```java
    final void callHandlerAdded() throws Exception {
        // 使用自旋方式，把状态设置为 ADD_COMPLETE，表示添加成功
        if (setAddComplete()) {
            // 调用 ChannelInitializer 的 handlerAdded 方法
            handler().handlerAdded(this);
        }
    }
```
