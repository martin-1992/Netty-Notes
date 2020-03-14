### NioEventLoopGroup#newChild

- 保存线程工厂 executor，在 [new ThreadPerTaskExecutor(newDefaultThreadFactory())](https://github.com/martin-1992/Netty-Notes/blob/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%88%9B%E5%BB%BA/ThreadPerTaskExecutor.md) 创建；
- 创建任务队列 taskQueue；
- 创建轮询器 Selector，Channel（包装过的 Socket）注册到该轮询器，具体注册流程可看 [Netty 服务端启动过程#initAndRegister](https://github.com/martin-1992/Netty-Notes/blob/master/Netty%20%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/register.md)。

```java
    @Override
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        return new NioEventLoop(this, executor, (SelectorProvider) args[0],
            ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
    }
```

### NioEventLoop 的构造函数
　　**使用 [openSelector](https://github.com/martin-1992/Netty-Notes/blob/master/NioEventLoop/NioEventLoop%20%E7%9A%84%E5%88%9B%E5%BB%BA/NioEventLoop%20%E7%9A%84%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0.md)，调用底层 JDK 创建 Selector。** 一个 Selector 绑定一个线程 NioEventLoop，一个 Selector 下有多个注册的 Channel（包装的 Socket）。

```java
    NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
                 SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler) {
        // 父类构造函数，在 SingleThreadEventExecutor 类中会初始化一个任务队列，其值为 DEFAULT_MAX_PENDING_TASKS
        super(parent, executor, false, DEFAULT_MAX_PENDING_TASKS, rejectedExecutionHandler);
        if (selectorProvider == null) {
            throw new NullPointerException("selectorProvider");
        }
        if (strategy == null) {
            throw new NullPointerException("selectStrategy");
        }
        provider = selectorProvider;
        // 一个 Selector 绑定一个线程 NioEventLoop，一个 Selector 下有多个 Channel（包装的 Socket）
        final SelectorTuple selectorTuple = openSelector();
        selector = selectorTuple.selector;
        unwrappedSelector = selectorTuple.unwrappedSelector;
        selectStrategy = strategy;
    }
```

### SingleThreadEventLoop
　　调用父类 SingleThreadEventExecutor。

```java
    protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor,
                                    boolean addTaskWakesUp, int maxPendingTasks,
                                    RejectedExecutionHandler rejectedExecutionHandler) {
        super(parent, executor, addTaskWakesUp, maxPendingTasks, rejectedExecutionHandler);
        tailTasks = newTaskQueue(maxPendingTasks);
    }
```

### SingleThreadEventExecutor
　　保存线程执行器，创建任务队列，用于保存异步任务。

```java
    protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                        boolean addTaskWakesUp, int maxPendingTasks,
                                        RejectedExecutionHandler rejectedHandler) {
        super(parent);
        this.addTaskWakesUp = addTaskWakesUp;
        // 任务队列的最大容量
        this.maxPendingTasks = Math.max(16, maxPendingTasks);
        // 保存线程工厂 executor
        this.executor = ObjectUtil.checkNotNull(executor, "executor");
        // 任务队列
        taskQueue = newTaskQueue(this.maxPendingTasks);
        // 拒绝策略，当任务队列满了后，无法添加新任务
        rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
    }
```
