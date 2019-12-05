### Worker
　　执行定时任务。

### run
　　检查，并执行定时任务。

- 唤醒阻塞在 start() 的线程，因为只允许一个线程执行，所以不用同步方法；
- 需要 sleep 到下一个格子检查定时任务的时间；
- 从任务队列中获取定时任务，根据定时时间计算格子索引，存入对应的格子中；
- 遍历该格子，执行定时任务；
- 移到下个格子，重复第二步到第三步，先 sleep 到下个格子的时间。（这里有可能执行任务过久，导致超过了下个格子的时间）；
- 遍历执行完所有格子的定时任务后，将每个格子中未过期和未取消的定时任务重新添加到待处理的任务队列 unprocessedTimeouts；
- 从定时任务队列中获取定时任务添加到待处理的任务队列 unprocessedTimeouts；
- 从格子中移除已取消的定时任务，

```java
    // 待处理（执行）的定时任务集合
    private final Set<Timeout> unprocessedTimeouts = new HashSet<Timeout>();

    // 格子计数，加一即移到下个格子
    private long tick;

    @Override
    public void run() {
        startTime = System.nanoTime();
        if (startTime == 0) {
            startTime = 1;
        }

        // 唤醒阻塞在 start() 的线程
        startTimeInitialized.countDown();

        do {
            // 需要 sleep 到下一个格子检查定时任务的时间
            final long deadline = waitForNextTick();
            if (deadline > 0) {
                // 获取格子索引，因为 tick 是一直往前推移（加一的），mask 是 2 的次方值，
                // 使用位运算 & 来获取索引值
                int idx = (int) (tick & mask);
                // 从格子中移除已取消的定时任务
                processCancelledTasks();
                // 根据格子索引获取对应的格子，即定时任务的链表
                HashedWheelBucket bucket =
                        wheel[idx];
                // 从任务队列中获取定时任务，根据定时时间计算格子索引，存入对应的格子中
                transferTimeoutsToBuckets();
                // 遍历该格子，执行定时任务
                bucket.expireTimeouts(deadline);
                // 移到下个格子
                tick++;
            }
        } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);

        // Fill the unprocessedTimeouts so we can return them from stop() method.
        // 遍历时间轮，将每个格子中未过期和未取消的定时任务重新添加到待处理的任务队列 unprocessedTimeouts
        for (HashedWheelBucket bucket: wheel) {
            bucket.clearTimeouts(unprocessedTimeouts);
        }
        // 从定时任务队列中获取定时任务添加到待处理的任务队列 unprocessedTimeouts
        for (;;) {
            HashedWheelTimeout timeout = timeouts.poll();
            // 定时任务队列为空
            if (timeout == null) {
                break;
            }
            // 定时任务未取消
            if (!timeout.isCancelled()) {
                unprocessedTimeouts.add(timeout);
            }
        }
        // 从格子中移除已取消的定时任务，因为任务是双向链表，所以使用基本的链表删除
        // 操作即可，不需要遍历
        processCancelledTasks();
    }
```


### transferTimeoutsToBuckets
　　从任务队列中获取定时任务，根据定时时间计算格子索引，存入对应的格子中。

```java
    private void transferTimeoutsToBuckets() {
        // transfer only max. 100000 timeouts per tick to prevent a thread to stale the workerThread when it just
        // adds new timeouts in a loop.
        // 每次只添加 100000 个任务到该格子中，防止一直添加任务，阻塞线程
        for (int i = 0; i < 100000; i++) {
            // 从定时任务队列中获取定时任务
            HashedWheelTimeout timeout = timeouts.poll();
            // 定时任务队列为空
            if (timeout == null) {
                // all processed
                break;
            }
            // 定时任务已取消，跳过
            if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) {
                // Was cancelled in the meantime.
                continue;
            }

            // 根据定时时间，计算需要经过多少个格子，比如定时时间为 700，一个格子的时间刻
            // 度 tickDuration 为 20，则需要 700 / 20 = 35，即经过 35 个格子
            long calculated = timeout.deadline / tickDuration;
            // 计算需要绕多少轮，假设一个时间轮有 20 个格子，即 wheel.length = 20，tick 为
            // 当前的格子，假设为 3，则 （35 - 3） / 20，还需要绕一轮
            timeout.remainingRounds = (calculated - tick) / wheel.length;
            // 如果该任务放在任务队列 timeouts 太久，过了执行时间，就放入当前的格子执行
            final long ticks = Math.max(calculated, tick); // Ensure we don't schedule for past.
            // 获取格子
            int stopIndex = (int) (ticks & mask);
            HashedWheelBucket bucket = wheel[stopIndex];
            // 将该定时任务添加到格子中
            bucket.addTimeout(timeout);
        }
    }
```

### processCancelledTasks
　　存放取消的定时任务，将这些任务取出，并从格子中移除这些已取消的定时任务，分两次进行。

- 第一次是使用 CAS 标记任务为已取消状态，并添加到一个专门存放取消任务的队列中；
- 第二次是从队列中获取这些已取消的任务，从格子中移除这些任务。

```java
    private void processCancelledTasks() {
        for (;;) {
            HashedWheelTimeout timeout = cancelledTimeouts.poll();
            if (timeout == null) {
                // all processed
                break;
            }
            try {
                // 从该格子中移除定时任务
                timeout.remove();
            } catch (Throwable t) {
                if (logger.isWarnEnabled()) {
                    logger.warn("An exception was thrown while process a cancellation task", t);
                }
            }
        }
    }
```

### waitForNextTick
　　需要 sleep 到下一个格子检查定时任务的时间。

```java
    private long waitForNextTick() {
        // tickDuration 为时间轮的基本时间跨度，tick 表示格子计数，假设 tickDuration 为 20，tick = 5（第六个格子），
        // tickDuration * (tick + 1) 表示需要 sleep 到下一次检查定时任务的时间
        long deadline = tickDuration * (tick + 1);

        for (;;) {
            // 获取当前时间（相对的），截止时间减去当前时间，表示还需要 sleep 的时间，除以 1000000 转为毫秒，这里
            // 加上 999999，为四舍五入，比如 deadline - currentTime=2000007，转为毫秒则为 2ms，但还缺 7 到达 
            // deadline 时间点，所以加上 999999，多睡一毫秒
            final long currentTime = System.nanoTime() - startTime;
            long sleepTimeMs = (deadline - currentTime + 999999) / 1000000;

            if (sleepTimeMs <= 0) {
                // 溢出问题，导致 sleepTimeMs 由正数变为负数
                if (currentTime == Long.MIN_VALUE) {
                    return -Long.MAX_VALUE;
                } else {
                    return currentTime;
                }
            }

            if (PlatformDependent.isWindows()) {
                sleepTimeMs = sleepTimeMs / 10 * 10;
                if (sleepTimeMs == 0) {
                    sleepTimeMs = 1;
                }
            }

            try {
                // 线程睡眠到下一个格子，再检查下个格子的定时任务
                Thread.sleep(sleepTimeMs);
            } catch (InterruptedException ignored) {
                if (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_SHUTDOWN) {
                    return Long.MIN_VALUE;
                }
            }
        }
    }
```
