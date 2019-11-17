
### DefaultChannelPipeline#read
　　调用头节点 headContext 的 read 方法，判断该 Channel 是否没有感兴趣的事件，是则向 Selector 注册对读事件感兴趣。

```java
        @Override
        public void read(ChannelHandlerContext ctx) {
            unsafe.beginRead();
        }
```


### AbstractChannel#beginRead
　　doBeginRead，核心逻辑。

```java
        @Override
        public final void beginRead() {
            assertEventLoop();

            if (!isActive()) {
                return;
            }

            try {
                doBeginRead();
            } catch (final Exception e) {
                invokeLater(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.fireExceptionCaught(e);
                    }
                });
                close(voidPromise());
            }
        }
```

### AbstractNioChannel#doBeginRead
　　如果 interestOps=0，表示没有感兴趣的事件，则向 Selector 注册对读事件感兴趣。

```java
    protected void doBeginRead() throws Exception {
        // Channel.read() or ChannelHandlerContext.read() was called
        final SelectionKey selectionKey = this.selectionKey;
        if (!selectionKey.isValid()) {
            return;
        }

        readPending = true;

        final int interestOps = selectionKey.interestOps();
        if ((interestOps & readInterestOp) == 0) {
            selectionKey.interestOps(interestOps | readInterestOp);
        }
    }
```
