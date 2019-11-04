### SelectStrategy
　　选择策略接口。

```java
public interface SelectStrategy {

    /**
     * 阻塞 select 策略
     */
    int SELECT = -1;
    
    /**
     * 重试策略
     */
    int CONTINUE = -2;、
    
    /**
     * 在轮询新事件
     */
    int BUSY_WAIT = -3;

    int calculateStrategy(IntSupplier selectSupplier, boolean hasTasks) throws Exception;
}
```


### DefaultSelectStrategy
　　SelectStrategy 的默认实现类。

- hasTasks 为 true，表示任务队列中有任务，使用 IntSupplier#get() 方法，该方法为接口方法，这里实现为 NioEventLoop 的 get 方法，无阻塞返回当前 Channel 新增的感兴趣的就绪 IO 事件数量；
- hasTasks 为 false，返回 -1，阻塞 select 策略。

```java
final class DefaultSelectStrategy implements SelectStrategy {
    // 单例模式
    static final SelectStrategy INSTANCE = new DefaultSelectStrategy();

    private DefaultSelectStrategy() { }

    @Override
    public int calculateStrategy(IntSupplier selectSupplier, boolean hasTasks) throws Exception {
        return hasTasks ? selectSupplier.get() : SelectStrategy.SELECT;
    }
}
```


### NioEventLoop#get
　　调用 JDK 底层的 selectNow 方法，无阻塞返回 Channel 新增的感兴趣的就绪 IO 事件数量。

```java
    private final IntSupplier selectNowSupplier = new IntSupplier() {
        @Override
        public int get() throws Exception {
            return selectNow();
        }
    };
    
    int selectNow() throws IOException {
        try {
            return selector.selectNow();
        } finally {
            // restore wakeup state if needed
            if (wakenUp.get()) {
                selector.wakeup();
            }
        }
    }
```

