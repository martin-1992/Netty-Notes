### start
　　启动时间轮。

```java
    // 
    public void start() {
        // 获取时间轮的状态
        switch (WORKER_STATE_UPDATER.get(this)) {
            case WORKER_STATE_INIT:
                // 时间轮为初始化，则启动 worker 线程来启动时间轮，使用 CAS 来更新状态，保证线程安全
                if (WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
                    // 在构造函数中会创建 worker 线程，然后启动
                    workerThread.start();
                }
                break;
            case WORKER_STATE_STARTED:
                // 时间轮已启动，则跳过
                break;
            case WORKER_STATE_SHUTDOWN:
                // 时间轮已关闭，则抛出异常
                throw new IllegalStateException("cannot be started once stopped");
            default:
                throw new Error("Invalid WorkerState");
        }

        // Wait until the startTime is initialized by the worker.
        // 等待 worker 线程初始化时间轮的启动时间
        while (startTime == 0) {
            try {
                startTimeInitialized.await();
            } catch (InterruptedException ignore) {
                // Ignore - it will be ready very soon.
            }
        }
    }
```
