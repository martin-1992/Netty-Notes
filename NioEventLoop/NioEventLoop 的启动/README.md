### NioEventLoop#run
　　run 方法主要做了三件事：

- [NioEventLoop#select](https://github.com/martin-1992/Netty-Notes/blob/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%90%AF%E5%8A%A8/select()%20%E6%96%B9%E6%B3%95%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.md)，检查是否有就绪的感兴趣的 IO 事件。调用 select 方法，轮询注册到 selector 上的 Channel，返回那些感兴趣的已就绪的 IO 事件的 Channel。Netty 会通过阻塞时间长短来判断是否触发空轮询 bug，如果当前阻塞一个 selector 操作，实际没有花这么长时间，有可能触发空轮询 bug，默认情况下，这个现象达到 512 次，则重建一个 selector，把之前 selector 上面所有的 key 重新移交到新的 selector，通过这种方式来避免 JDK 空轮询 bug 的；
- [NioEventLoop#processSelectedKeys](https://github.com/martin-1992/Netty-Notes/blob/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%90%AF%E5%8A%A8/processSelectedKeys.md)，通过 JDK 底层 API 获取注册到该 Selector 的所有 Channel 中已就绪的 IO 事件，通过遍历 selectedKeys 处理。如果当前线程为 EventLoop 线程，则直接执行 IO 任务，不会放到任务队列中，通过该方法异步执行 selectionKey 中 ready 的事件，比如 accept、connect、read、write 等。这里不在 EventLoop 线程会直接返回，而在 pipeline 中如果不在 EventLoop 线程，则会使用该线程的线程工厂 Executor 创建一个线程任务放到任务队列中执行；
- [SingleThreadEventExecutora#runAllTasks](https://github.com/martin-1992/Netty-Notes/blob/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%90%AF%E5%8A%A8/runAllTasks.md)，该方法有两个，一个不带参数，是执行普通任务队列的任务直到全部任务执行完毕。另一个是带超时时间的方法，**使用时间片方式，来轮询处理 Channel 中的读写事件和任务队列中的事件。** 防止处理任务队列的时间太长，导致没法处理 Channel 中的读写事件。两个方法都会将到指定时间的定时任务添加到普通任务队列中执行，如果当前线程不为 NioEventLoop 线程，则将任务添加到任务队列中执行，比如注册 register0、重建 Selector（解决空轮询 bug），然后调用该方法执行。

```java
    @Override
    protected void run() {
        for (;;) {
            try {
                try {
                    // 轮询 IO 事件，一个 Selector 对应一个 NioEventLoop，这个 Select 方法轮询注册到 selector 上面的 IO 事件
                    switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:

                    case SelectStrategy.SELECT:
                        // 每次进行 select 操作，使用 CAS 其设为 false，未唤醒状态
                        select(wakenUp.getAndSet(false));

                        // 判断是否需要唤醒，是则唤醒 selector 线程
                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                        // fall through
                    default:
                    }
                } catch (IOException e) {
                    // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                    // the selector and retry. https://github.com/netty/netty/issues/8566
                    rebuildSelector0();
                    handleLoopException(e);
                    continue;
                }

                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                if (ioRatio == 100) {
                    try {
                        // 处理 IO 事件
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        // 处理异步队列的事件
                        runAllTasks();
                    }
                } else {
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                // 关闭
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
```

