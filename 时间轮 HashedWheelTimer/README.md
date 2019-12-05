### 构造函数

- [createWheel](https://github.com/martin-1992/Netty-Notes/blob/master/%E6%97%B6%E9%97%B4%E8%BD%AE%20HashedWheelTimer/createWheel.md)，创建时间轮，在 newTimeout 方法中启动；
- threadFactory.newThread(worker)，创建 [worker](https://github.com/martin-1992/Netty-Notes/blob/master/%E6%97%B6%E9%97%B4%E8%BD%AE%20HashedWheelTimer/Worker.md) 线程，在 newTimeout 方法中会启动 worker 线程，遍历时间轮的所有格子，执行定时任务。

```java
    public HashedWheelTimer(
            // 线程工厂，创建 worker 线程
            ThreadFactory threadFactory,
            // 时间轮的基本时间跨度，即指针多久转一格
            long tickDuration,
            // tickDuration 的时间单位
            TimeUnit unit,
            // 时间轮的时间个数，即一圈有多少格
            int ticksPerWheel,
            // 是否开启内存泄漏检测
            boolean leakDetection,
            // 最大待处理的定时任务数
            long maxPendingTimeouts) {

        if (threadFactory == null) {
            throw new NullPointerException("threadFactory");
        }
        if (unit == null) {
            throw new NullPointerException("unit");
        }
        if (tickDuration <= 0) {
            throw new IllegalArgumentException("tickDuration must be greater than 0: " + tickDuration);
        }
        if (ticksPerWheel <= 0) {
            throw new IllegalArgumentException("ticksPerWheel must be greater than 0: " + ticksPerWheel);
        }

        // Normalize ticksPerWheel to power of two and initialize the wheel.
        // 创建时间轮，用于存储定时任务的环形队列，底层用数组实现
        wheel = createWheel(ticksPerWheel);
        // 用于计算时间轮格子的索引
        mask = wheel.length - 1;

        // Convert tickDuration to nanos.
        // 转换成纳秒
        long duration = unit.toNanos(tickDuration);

        // Prevent overflow.
        // 防止溢出，格子的时间跨度 * 格子数不能大于最大值，即 duration * wheel.length >= Long.MAX_VALUE
        if (duration >= Long.MAX_VALUE / wheel.length) {
            throw new IllegalArgumentException(String.format(
                    "tickDuration: %d (expected: 0 < tickDuration in nanos < %d",
                    tickDuration, Long.MAX_VALUE / wheel.length));
        }

        // 格子的时间跨度太小，使用 MILLISECOND_NANOS
        if (duration < MILLISECOND_NANOS) {
            logger.warn("Configured tickDuration {} smaller then {}, using 1ms.",
                        tickDuration, MILLISECOND_NANOS);
            this.tickDuration = MILLISECOND_NANOS;
        } else {
            this.tickDuration = duration;
        }

        // 创建 worker 线程
        workerThread = threadFactory.newThread(worker);

        // 默认启动内存检测
        leak = leakDetection || !workerThread.isDaemon() ? leakDetector.track(this) : null;
        // 最大待处理的定时任务数
        this.maxPendingTimeouts = maxPendingTimeouts;

        if (INSTANCE_COUNTER.incrementAndGet() > INSTANCE_COUNT_LIMIT &&
            WARNED_TOO_MANY_INSTANCES.compareAndSet(false, true)) {
            reportTooManyInstances();
        }
    }
```

### newTimeout
　　启动 [start](https://github.com/martin-1992/Netty-Notes/blob/master/%E6%97%B6%E9%97%B4%E8%BD%AE%20HashedWheelTimer/start.md) 时间轮，计算定时任务的定时时间，将定时任务包装成 [HashedWheelTimeout](https://github.com/martin-1992/Netty-Notes/blob/master/%E6%97%B6%E9%97%B4%E8%BD%AE%20HashedWheelTimer/HashedWheelTimeout.md) 类，添加到定时任务队列，由 work 线程从定时任务队列中添加到对应的格子中。

```java
    @Override
    public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        if (unit == null) {
            throw new NullPointerException("unit");
        }

        // 定时任务数加一
        long pendingTimeoutsCount = pendingTimeouts.incrementAndGet();
        // 定时任务数超过最大限额，抛出异常
        if (maxPendingTimeouts > 0 && pendingTimeoutsCount > maxPendingTimeouts) {
            pendingTimeouts.decrementAndGet();
            throw new RejectedExecutionException("Number of pending timeouts ("
                + pendingTimeoutsCount + ") is greater than or equal to maximum allowed pending "
                + "timeouts (" + maxPendingTimeouts + ")");
        }
        // 启动时间轮
        start();

        // Add the timeout to the timeout queue which will be processed on the next tick.
        // During processing all the queued HashedWheelTimeouts will be added to the correct HashedWheelBucket.
        // 计算定时任务的定时时间（相对的）
        long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;

        // Guard against overflow.
        // 防止溢出
        if (delay > 0 && deadline < 0) {
            deadline = Long.MAX_VALUE;
        }
        // 将定时任务包装成 HashedWheelTimeout 类，它是一个节点，能添加到 HashedWheelBucket 链表中
        HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
        // 先将待执行的定时任务添加到队列 timeouts 中，当 worker 线程启动，检查格子
        // 时，会将（最多 10000 个）任务添加到对应的格子中，然后执行
        timeouts.add(timeout);
        return timeout;
    }
```


