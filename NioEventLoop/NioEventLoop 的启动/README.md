### NioEventLoop#run
　　run 方法主要做了三件事：

- [NioEventLoop#select]()，检查是否有 IO 事件。调用 select 方法，轮询注册到 selector 上面的连接 IO 事件。Netty 会通过阻塞时间长短来判断是否触发空轮询 bug，如果当前阻塞一个 selector 操作，实际没有花这么长时间，有可能触发空轮询 bug，默认情况下，这个现象达到 512 次，则重建一个 selector，把之前 selector 上面所有的 key 重新移交到新的 selector，通过这种方式来避免 JDK 空轮询 bug 的；
- [NioEventLoop#processSelectedKeys]()，处理 IO 事件，处理在前面轮询出来的 IO 事件；
- [SingleThreadEventExecutora#runAllTasks]()，处理异步任务队列，处理外部线程扔到 task 里面的任务。通过 inEventLoop 方法来判断是否为外部线程，是则将所有操作封装成一个 task，放到普通队列中 MPScQueue，然后在 runAllTasks() 里会处理执行队列中的任务操作。

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

