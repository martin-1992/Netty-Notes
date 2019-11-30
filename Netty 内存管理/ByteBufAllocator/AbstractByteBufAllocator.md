### AbstractByteBufAllocator
　　AbstractByteBufAllocator 为 ByteBufAllocator 的默认实现类。

```java
public abstract class AbstractByteBufAllocator implements ByteBufAllocator {
    static final int DEFAULT_INITIAL_CAPACITY = 256;
    static final int DEFAULT_MAX_CAPACITY = Integer.MAX_VALUE;
    static final int DEFAULT_MAX_COMPONENTS = 16;
    static final int CALCULATE_THRESHOLD = 1048576 * 4; // 4 MiB page
```


### buffer
　　如果 directByDefault 为 true，则分配堆内 directBuffer，否则分配堆外 heapBuffer。

```java
    @Override
    public ByteBuf buffer() {
        if (directByDefault) {
            return directBuffer();
        }
        return heapBuffer();
    }

    @Override
    public ByteBuf buffer(int initialCapacity) {
        if (directByDefault) {
            return directBuffer(initialCapacity);
        }
        return heapBuffer(initialCapacity);
    }

    @Override
    public ByteBuf buffer(int initialCapacity, int maxCapacity) {
        if (directByDefault) {
            return directBuffer(initialCapacity, maxCapacity);
        }
        return heapBuffer(initialCapacity, maxCapacity);
    }
```


#### directBuffer
　　最终调用 newDirectBuffer，为抽象方法，即堆内分配交由子类实现的。

```java
    @Override
    public ByteBuf directBuffer() {
        return directBuffer(DEFAULT_INITIAL_CAPACITY, DEFAULT_MAX_CAPACITY);
    }
    
    @Override
    public ByteBuf directBuffer(int initialCapacity, int maxCapacity) {
        // 参数校验
        if (initialCapacity == 0 && maxCapacity == 0) {
            return emptyBuf;
        }
        // 初始容量需小于最大容纳量
        validate(initialCapacity, maxCapacity);
        return newDirectBuffer(initialCapacity, maxCapacity);
    }
    
    protected abstract ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity);
```


#### heapBuffer
　　同理，heapBuffer 的 newHeapBuffer 方法由子类实现。

```java
    protected abstract ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity);
```


### calculateNewCapacity

```java
    @Override
    public int calculateNewCapacity(int minNewCapacity, int maxCapacity) {
        checkPositiveOrZero(minNewCapacity, "minNewCapacity");
        // 参数校验
        if (minNewCapacity > maxCapacity) {
            throw new IllegalArgumentException(String.format(
                    "minNewCapacity: %d (expected: not greater than maxCapacity(%d)",
                    minNewCapacity, maxCapacity));
        }
        final int threshold = CALCULATE_THRESHOLD; // 4 MiB page

        if (minNewCapacity == threshold) {
            return threshold;
        }

        // If over threshold, do not double but just increase by threshold.
        if (minNewCapacity > threshold) {
            int newCapacity = minNewCapacity / threshold * threshold;
            if (newCapacity > maxCapacity - threshold) {
                newCapacity = maxCapacity;
            } else {
                newCapacity += threshold;
            }
            return newCapacity;
        }

        // Not over threshold. Double up to 4 MiB, starting from 64.
        int newCapacity = 64;
        while (newCapacity < minNewCapacity) {
            newCapacity <<= 1;
        }

        return Math.min(newCapacity, maxCapacity);
    }
```

