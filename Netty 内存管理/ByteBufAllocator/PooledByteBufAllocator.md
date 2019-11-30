### PooledByteBufAllocator
　　基于内存池的 ByteBuf 的分配器。

### newDirectBuffer

- threadCache.get()，因为调用 newDirectBuffer 可能是多线程的，所以会使用类似 ThreadLocal 的机制，即 PoolThreadLocalCache 来获取线程局部缓存；
- cache.directArena，从线程局部缓存中获取当前线程的 PoolArena。在 PooledByteBufAllocator#initialValue 时会创建 PoolArena；
- directArena.allocate，内存分配。

```java
    private final PoolThreadLocalCache threadCache;

    @Override
    protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        PoolThreadCache cache = threadCache.get();
        PoolArena<ByteBuffer> directArena = cache.directArena;

        final ByteBuf buf;
        if (directArena != null) {
            buf = directArena.allocate(cache, initialCapacity, maxCapacity);
        } else {
            buf = PlatformDependent.hasUnsafe() ?
                    UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity) :
                    new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
        }

        return toLeakAwareBuffer(buf);
    }
```
