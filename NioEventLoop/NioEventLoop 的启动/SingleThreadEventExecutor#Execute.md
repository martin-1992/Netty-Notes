### SingleThreadEventExecutor#execute

- 将任务添加到任务队列中，等待执行；
- 如果不在 EventLoop 线程中，即外部线程执行 Netty 的任务，则创建一个线程执行，不阻塞当前线程。

```java
    @Override
    public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        // 判断当前执行的线程是否为 EventLoop 的线程
        boolean inEventLoop = inEventLoop();
        // 将该任务添加到任务队列中等待执行
        addTask(task);
        // 不在 EventLoop 线程中，则创建一个线程执行，不阻塞当前线程
        if (!inEventLoop) {
            startThread();
            if (isShutdown()) {
                // 已关闭，则从任务队列中移除该任务
                boolean reject = false;
                try {
                    if (removeTask(task)) {
                        reject = true;
                    }
                } catch (UnsupportedOperationException e) {
                    // The task queue does not support removal so the best thing we can do is to just move on and
                    // hope we will be able to pick-up the task before its completely terminated.
                    // In worst case we will log on termination.
                }
                if (reject) {
                    reject();
                }
            }
        }
        // addTaskWakesUp 表示添加是否会唤醒 EventLoop 线程，不会则使用主动唤醒
        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
```

