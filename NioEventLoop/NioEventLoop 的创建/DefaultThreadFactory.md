### DefaultThreadFactory
　　DefaultThreadFactory 用于创建新线程的线程工厂。

### 构造函数
　　创建线程工厂 ThreadFactory，定义线程名称规则，并配置线程参数，包括是否为守护线程、线程的优先级等。

```java
    public DefaultThreadFactory(String poolName, boolean daemon, int priority, ThreadGroup threadGroup) {
        if (poolName == null) {
            throw new NullPointerException("poolName");
        }
        // 线程优先级参数校验
        if (priority < Thread.MIN_PRIORITY || priority > Thread.MAX_PRIORITY) {
            throw new IllegalArgumentException(
                    "priority: " + priority + " (expected: Thread.MIN_PRIORITY <= priority <= Thread.MAX_PRIORITY)");
        }
        // 线程名称的前缀，比如 xxx-0-，poolId.incrementAndGet() 表示每创建一个 EventLoopGroup 则自增
        prefix = poolName + '-' + poolId.incrementAndGet() + '-';
        this.daemon = daemon;
        this.priority = priority;
        this.threadGroup = threadGroup;
    }
```

### toPoolName
　　对类名进行处理，将首字母转为小写，比如 NioEventLoop -> nioEventLoop，用于定义线程名。

```java
    public static String toPoolName(Class<?> poolType) {
        if (poolType == null) {
            throw new NullPointerException("poolType");
        }
        // 获取类名，比如 NioEventLoop
        String poolName = StringUtil.simpleClassName(poolType);
        switch (poolName.length()) {
            case 0:
                return "unknown";
            case 1:
                return poolName.toLowerCase(Locale.US);
            default:
                if (Character.isUpperCase(poolName.charAt(0)) && Character.isLowerCase(poolName.charAt(1))) {
                    return Character.toLowerCase(poolName.charAt(0)) + poolName.substring(1);
                } else {
                    return poolName;
                }
        }
    }
```


### newThread
　　使用线程工厂创建新线程，线程名为类名-线程组自增 ID-线程自增 ID，配置新线程的参数，最后会调用底层调用 JDK 的 thread，将其封装为 FastThreadLocalThread。

```java
    @Override
    public Thread newThread(Runnable r) {
        // 创建新线程，prefix 为 xxx-1-，加上线程自增 ID nextId.incrementAndGet()，
        // 格式为类名-线程组自增 ID-线程自增 ID，比如 nioEventLoop-1-1
        Thread t = newThread(FastThreadLocalRunnable.wrap(r), prefix + nextId.incrementAndGet());
        try {
            // 配置是否为守护线程
            if (t.isDaemon() != daemon) {
                t.setDaemon(daemon);
            }
            // 配置线程的优先级
            if (t.getPriority() != priority) {
                t.setPriority(priority);
            }
        } catch (Exception ignored) {
            // Doesn't matter even if failed to set.
        }
        return t;
    }
    
    protected Thread newThread(Runnable r, String name) {
        // 底层调用 JDK 的 thread
        return new FastThreadLocalThread(threadGroup, r, name);
    }
```

