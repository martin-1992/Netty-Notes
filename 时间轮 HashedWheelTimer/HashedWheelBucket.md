### HashedWheelBucket
　　根据时间轮的格子数 ticksPerWheel 创建 HashedWheelBucket 数组，HashedWheelBucket 为一个双向链表。

### addTimeout
　　将定时任务插入到链表末尾。

```java
        public void addTimeout(HashedWheelTimeout timeout) {
            assert timeout.bucket == null;
            timeout.bucket = this;
            if (head == null) {
                head = tail = timeout;
            } else {
                tail.next = timeout;
                timeout.prev = tail;
                tail = timeout;
            }
        }
```

### remove
　　从链表中删除该定时任务。

```java
    public HashedWheelTimeout remove(HashedWheelTimeout timeout) {
        HashedWheelTimeout next = timeout.next;
        // remove timeout that was either processed or cancelled by updating the linked-list
        if (timeout.prev != null) {
            timeout.prev.next = next;
        }
        if (timeout.next != null) {
            timeout.next.prev = timeout.prev;
        }

        if (timeout == head) {
            // if timeout is also the tail we need to adjust the entry too
            if (timeout == tail) {
                tail = null;
                head = null;
            } else {
                head = next;
            }
        } else if (timeout == tail) {
            // if the timeout is the tail modify the tail to be the prev node.
            tail = timeout.prev;
        }
        // null out prev, next and bucket to allow for GC.
        timeout.prev = null;
        timeout.next = null;
        timeout.bucket = null;
        timeout.timer.pendingTimeouts.decrementAndGet();
        return next;
    }
```

### expireTimeouts
　　 [HashedWheelTimeout#expire]()，使用 CAS 设置定时任务状态为已过期，并调用 worker 线程执行定时任务。

```java
    public void expireTimeouts(long deadline) {
        HashedWheelTimeout timeout = head;
    
        // process all timeouts
        // 遍历该格子的所有定时任务
        while (timeout != null) {
            HashedWheelTimeout next = timeout.next;
            // 定时任务到期
            if (timeout.remainingRounds <= 0) {
                // 从链表中删除该定时任务，返回下个定时任务
                next = remove(timeout);
                // 使用 CAS 设置为已到期
                if (timeout.deadline <= deadline) {
                    timeout.expire();
                } else {
                    // The timeout was placed into a wrong slot. This should never happen.
                    // 定时任务已到期，remainingRounds 小于等于 0，但定时任务的 deadline 大于指定的 deadline，
                    // 表示定时任务放错格子，这种情况不会出现。
                    throw new IllegalStateException(String.format(
                            "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
                }
            } else if (timeout.isCancelled()) {
                // 定时任务已取消
                next = remove(timeout);
            } else {
                // 定时任务没到期，轮数减一
                timeout.remainingRounds --;
            }
            // 设置下个定时任务 next 为 timeout，继续遍历
            timeout = next;
        }
    }
```

### clearTimeouts
　　清除该各自的所有定时任务，将未过期和未取消的定时任务添加到 set 中。

```java
    public void clearTimeouts(Set<Timeout> set) {
        for (;;) {
            // 获取定时任务
            HashedWheelTimeout timeout = pollTimeout();
            if (timeout == null) {
                return;
            }
            if (timeout.isExpired() || timeout.isCancelled()) {
                continue;
            }
            set.add(timeout);
        }
    }
```

#### pollTimeout
　　从链表中弹出头节点（定时任务）。

```java
    private HashedWheelTimeout pollTimeout() {
        HashedWheelTimeout head = this.head;
        if (head == null) {
            return null;
        }
        HashedWheelTimeout next = head.next;
        if (next == null) {
            tail = this.head =  null;
        } else {
            // 设置下个节点为新的头节点
            this.head = next;
            next.prev = null;
        }

        // null out prev and next to allow for GC.
        head.next = null;
        head.prev = null;
        head.bucket = null;
        return head;
    }
```

