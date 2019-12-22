### MemoryRegionCache
　　PoolThreadCache 的内部静态类，内存块缓存。维护一个 Entry 对象的队列，存储缓存的内存块。队列为 MPSC( Multiple Producer Single Consumer ) 队列，即多个生产者单一消费者。

```java
    private abstract static class MemoryRegionCache<T> {
        // 队列大小，默认 tiny 为 512、small 为 256、normal 为 64
        private final int size;
        // 存储每种大小的 ByteBuf
        private final Queue<Entry<T>> queue;
        // 为枚举类，有 tiny、small、normal
        private final SizeClass sizeClass;
        // 分配计数器，每次调用 allocate 方法时加一
        private int allocations;

        MemoryRegionCache(int size, SizeClass sizeClass) {
            // 对 size 进行规格化，需为 2 的幂次方，比如传入的 size 为 15，则更正为 16（2^8）
            this.size = MathUtil.safeFindNextPositivePowerOfTwo(size);
            // 创建队列，队列大小为 size
            queue = PlatformDependent.newFixedMpscQueue(this.size);
            // 该 MemoryRegionCache 的缓存类型为 tiny 或 small 或 normal
            this.sizeClass = sizeClass;
        }
    }
```

### Entry
　　在 chunk 上分配内存对象 handle。

```java
    static final class Entry<T> {
        // 只有一个 recycle 方法，用于回收对象
        final Handle<Entry<?>> recyclerHandle;
        PoolChunk<T> chunk;
        ByteBuffer nioBuffer;
        // 通过 handle 可唯一定位 chunk 的内存
        long handle = -1;

        Entry(Handle<Entry<?>> recyclerHandle) {
            this.recyclerHandle = recyclerHandle;
        }

        // 回收 Entry 对象
        void recycle() {
            chunk = null;
            nioBuffer = null;
            handle = -1;
            recyclerHandle.recycle(this);
        }
    }
```


### newEntry

- [Recycler#get](https://github.com/martin-1992/Netty-Notes/blob/master/Recycler/get.md)，从对象池 Recycler 中获取对象；
- 设置该对象的 chunk、nioBuffer 和 handle。

```java
    @SuppressWarnings("rawtypes")
    private static Entry newEntry(PoolChunk<?> chunk, ByteBuffer nioBuffer, long handle) {
        Entry entry = RECYCLER.get();
        entry.chunk = chunk;
        entry.nioBuffer = nioBuffer;
        entry.handle = handle;
        return entry;
    }
```
