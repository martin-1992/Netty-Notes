### UnpooledByteBufAllocator 
　　普通的 ByteBuf 的分配器，不基于内存池。

### 构造函数
　　
- metric，计数器，可获取堆内和堆外的内存使用情况；
- disableLeakDetector，是否禁用内存泄漏检测功能，默认为 false；
- noCleaner，是否不使用 cleaner 来释放直接内存，默认为 true。

```java
public final class UnpooledByteBufAllocator extends AbstractByteBufAllocator implements ByteBufAllocatorMetricProvider {

    private final UnpooledByteBufAllocatorMetric metric = new UnpooledByteBufAllocatorMetric();
    
    private final boolean disableLeakDetector;
    
    private final boolean noCleaner;

    public static final UnpooledByteBufAllocator DEFAULT =
            new UnpooledByteBufAllocator(PlatformDependent.directBufferPreferred());

    public UnpooledByteBufAllocator(boolean preferDirect) {
        this(preferDirect, false);
    }

    public UnpooledByteBufAllocator(boolean preferDirect, boolean disableLeakDetector) {
        this(preferDirect, disableLeakDetector, PlatformDependent.useDirectBufferNoCleaner());
    }

    public UnpooledByteBufAllocator(boolean preferDirect, boolean disableLeakDetector, boolean tryNoCleaner) {
        super(preferDirect);
        this.disableLeakDetector = disableLeakDetector;
        noCleaner = tryNoCleaner && PlatformDependent.hasUnsafe()
                && PlatformDependent.hasDirectBufferNoCleanerConstructor();
    }
} 
```

### newHeapBuffer
　　根据是否有 Unsafe 来判断，heap 内存的分配。

```java
    @Override
    protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
        return PlatformDependent.hasUnsafe() ?
                // heap 内存的分配，有 Unsafe
                new InstrumentedUnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
                // heap 内存的分配，非 Unsafe
                new InstrumentedUnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
    }
```

#### InstrumentedUnpooledUnsafeHeapByteBuf
　　分配内存，根据初始容量创建字节数组。调用 metric 增加内存。

```java
    private static final class InstrumentedUnpooledUnsafeHeapByteBuf extends UnpooledUnsafeHeapByteBuf {
        InstrumentedUnpooledUnsafeHeapByteBuf(UnpooledByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
            super(alloc, initialCapacity, maxCapacity);
        }

        @Override
        protected byte[] allocateArray(int initialCapacity) {
            // 分配内存，根据初始容量创建字节数组
            byte[] bytes = super.allocateArray(initialCapacity);
            // 增加堆内存的使用
            ((UnpooledByteBufAllocator) alloc()).incrementHeap(bytes.length);
            return bytes;
        }

        @Override
        protected void freeArray(byte[] array) {
            int length = array.length;
            // 释放内存
            super.freeArray(array);
            // 减少堆内存的使用
            ((UnpooledByteBufAllocator) alloc()).decrementHeap(length);
        }
    }
```

#### InstrumentedUnpooledHeapByteBuf
　　分配内存，根据初始容量创建字节数组。调用 metric 增加内存。

```java
    private static final class InstrumentedUnpooledHeapByteBuf extends UnpooledHeapByteBuf {
        InstrumentedUnpooledHeapByteBuf(UnpooledByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
            super(alloc, initialCapacity, maxCapacity);
        }

        @Override
        protected byte[] allocateArray(int initialCapacity) {
            // 分配内存，创建字节数
            byte[] bytes = super.allocateArray(initialCapacity);
            // 增加堆内存的使用
            ((UnpooledByteBufAllocator) alloc()).incrementHeap(bytes.length);
            return bytes;
        }

        @Override
        protected void freeArray(byte[] array) {
            int length = array.length;
            // 释放内存
            super.freeArray(array);
            // 减少堆内存的使用
            ((UnpooledByteBufAllocator) alloc()).decrementHeap(length);
        }
    }
```

### newDirectBuffer
　　根据是否有 Unsafe 来判断，direct 内存的分配。
  
```java
    @Override
    protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        final ByteBuf buf;
        if (PlatformDependent.hasUnsafe()) {
            buf = noCleaner ? new InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf(this, initialCapacity, maxCapacity) :
                    new InstrumentedUnpooledUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
        } else {
            buf = new InstrumentedUnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
        }
        return disableLeakDetector ? buf : toLeakAwareBuffer(buf);
    }
```

### metric
　　获取计数器，通过计数器可增加、减少 direct 和 heap 内存的值。

```java
    @Override
    public ByteBufAllocatorMetric metric() {
        return metric;
    }
    
    /**
     * Direct 内存使用增加
     * @param amount
     */
    void incrementDirect(int amount) {
        metric.directCounter.add(amount);
    }
    
    void decrementDirect(int amount) {
        metric.directCounter.add(-amount);
    }

    void incrementHeap(int amount) {
        metric.heapCounter.add(amount);
    }

    void decrementHeap(int amount) {
        metric.heapCounter.add(-amount);
    }
```
