### allocate
　　从队列中获取缓存的内存块，初始化到 PooledByteBuf 对象中，并返回是否分配成功。

- 从队列里弹出一个 Entry 对象（缓存的内存块）；
- initBuf，对 Entry 进行初始化，抽象方法，由子类 SubPageMemoryRegionCache 和 NormalMemoryRegionCache 来实现；
- 初始化完成后，对 Entry 进行回收，放回到对象池中。

```java
    @SuppressWarnings({ "unchecked", "rawtypes" })
    private boolean allocate(MemoryRegionCache<?> cache, PooledByteBuf buf, int reqCapacity) {
        if (cache == null) {
            // no cache found so just return false here
            return false;
        }

        boolean allocated = cache.allocate(buf, reqCapacity);
        if (++ allocations >= freeSweepAllocationThreshold) {
            allocations = 0;
            trim();
        }
        return allocated;
    }

    public final boolean allocate(PooledByteBuf<T> buf, int reqCapacity) {
        // 从队列里弹出一个元素
        Entry<T> entry = queue.poll();
        if (entry == null) {
            return false;
        }
        // 对 entry 进行初始化
        initBuf(entry.chunk, entry.nioBuffer, entry.handle, buf, reqCapacity);
        // 初始化完后，entry 则没用。这边 Netty 为了复用 entry，会将 entry 扔到对象池
        // 后续 ByteBuf 进行回收时，它直接从对象池里取一个 entry，然后把 entry 的 chunk
        // 和 handle 指向被回收的 ByteBuf，就可以直接复用了
        entry.recycle();

        // allocations is not thread-safe which is fine as this is only called from the same thread all time.
        ++ allocations;
        return true;
    }
```

