### HashedWheelTimeout
　　为定时任务的包装类，将定时任务包装成节点，添加到 HashedWheelBucket 链表中。

```java
    private static final class HashedWheelTimeout implements Timeout {

        // 定时任务状态，初始化、取消、过期
        private static final int ST_INIT = 0;
        private static final int ST_CANCELLED = 1;
        private static final int ST_EXPIRED = 2;

        private static final AtomicIntegerFieldUpdater<HashedWheelTimeout> STATE_UPDATER =
                AtomicIntegerFieldUpdater.newUpdater(HashedWheelTimeout.class, "state");

        private final HashedWheelTimer timer;
        // 定时任务
        private final TimerTask task;
        private final long deadline;
        
        // 定时任务的初始状态，使用 volatile 修饰，保证可见性
        @SuppressWarnings({"unused", "FieldMayBeFinal", "RedundantFieldInitialization" })
        private volatile int state = ST_INIT;

        // 离任务执行的轮数，当将次任务加入到格子中是计算该值，每过一轮，该值减一
        long remainingRounds;

        // 在 worker 线程，不需要使用同步 synchronized，双向链表结构
        HashedWheelTimeout next;
        HashedWheelTimeout prev;

        // 定时任务所在的格子
        HashedWheelBucket bucket;

        HashedWheelTimeout(HashedWheelTimer timer, TimerTask task, long deadline) {
            this.timer = timer;
            this.task = task;
            this.deadline = deadline;
        }

        /**
         * 获取时间轮
         **/
        @Override
        public Timer timer() {
            return timer;
        }

        /**
         * 获取定时任务
         **/
        @Override
        public TimerTask task() {
            return task;
        }
    }
```

### cancel
　　使用 CAS 设置定时任务的状态为已取消，并添加到 MPSC 队列，下次检查该格子的定时任务时，将其从格子中移除。

```java
    @Override
    public boolean cancel() {
        if (!compareAndSetState(ST_INIT, ST_CANCELLED)) {
            return false;
        }
        // 添加到 MPSC 队列
        timer.cancelledTimeouts.add(this);
        return true;
    }
```

### remove
　　从该格子中移除定时任务。

```java
    void remove() {
        HashedWheelBucket bucket = this.bucket;
        if (bucket != null) {
            bucket.remove(this);
        } else {
            timer.pendingTimeouts.decrementAndGet();
        }
    }
```

### expire
　　使用 CAS 设置定时任务状态为已过期，并执行定时任务。
     
```java
        public void expire() {
            if (!compareAndSetState(ST_INIT, ST_EXPIRED)) {
                return;
            }

            try {
                task.run(this);
            } catch (Throwable t) {
                if (logger.isWarnEnabled()) {
                    logger.warn("An exception was thrown by " + TimerTask.class.getSimpleName() + '.', t);
                }
            }
        }
```

