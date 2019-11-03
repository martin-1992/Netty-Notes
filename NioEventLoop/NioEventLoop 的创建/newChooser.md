### DefaultEventExecutorChooserFactory#newChooser
　　根据数组长度，判断创建数组的长度是否为 2 的次方，能否使用位运算获取索引下标。

```java
    @SuppressWarnings("unchecked")
    @Override
    public EventExecutorChooser newChooser(EventExecutor[] executors) {
        if (isPowerOfTwo(executors.length)) {
            return new PowerOfTwoEventExecutorChooser(executors);
        } else {
            return new GenericEventExecutorChooser(executors);
        }
    }
```


### isPowerOfTwo
　　是否为 2 的次方值，高位为 1，低位为 0，减去 1 后高位为 0，低位为 1，相与 & 后等于原来的值，则是 2 的次方值。比如 8=1000，1000-1=0111，1000 & 0111 = 1000，所以 8 为 2 的次方值。

```java
    private static boolean isPowerOfTwo(int val) {
        return (val & -val) == val;
    }
```


### PowerOfTwoEventExecutorChooser
　　如果数组长度为 2 的次方值，则可通过位运算来获取索引下标，速度更快。这里 idx 是自增的，当连接数超过线程数，会重新从 0 开始绑定，比如线程数为 8，第 9 个连接进来，则绑定第一个线程。

```java
    private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        PowerOfTwoEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            // 通过位预算获取 executor 的索引下标，速度更快
            return executors[idx.getAndIncrement() & executors.length - 1];
        }
    }
```


### GenericEventExecutorChooser
　　如果数组长度不为 2 的次方值，则使用正常方法获取索引下标。

```java
    private static final class GenericEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        GenericEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[Math.abs(idx.getAndIncrement() % executors.length)];
        }
    }
```

