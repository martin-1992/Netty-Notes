### allocate
　　从队列中获取缓存的内存块，初始化到 PooledByteBuf 对象中，并返回是否分配成功。

- 从队列里弹出一个 Entry 对象（缓存的内存块）；
- initBuf，对 Entry 进行初始化，抽象方法，由子类 SubPageMemoryRegionCache 和 NormalMemoryRegionCache 来实现；
- 初始化完成后，对 Entry 进行回收，放回到对象池中。

```java
public final boolean allocate(PooledByteBuf<T> buf, int reqCapacity) {
    // 从队列里弹出一个 Entry 对象
    Entry<T> entry = queue.poll();
    if (entry == null) {
        return false;
    }
    // 对 Entry 进行初始化，抽象方法，由子类 SubPageMemoryRegionCache 和 NormalMemoryRegionCache 来实现
    initBuf(entry.chunk, entry.nioBuffer, entry.handle, buf, reqCapacity);
    // 对 Entry 进行回收，放回到对象池中
    entry.recycle();

    ++ allocations;
    return true;
}

protected abstract void initBuf(PoolChunk<T> chunk, ByteBuffer nioBuffer, long handle,
                                PooledByteBuf<T> buf, int reqCapacity);
```

### PoolThreadCache#SubPageMemoryRegionCache
　　以 SubPageMemoryRegionCache 为例，继续调用 PoolChunk#initBufWithSubpage 的方法。

```java
@Override
protected void initBuf(
        PoolChunk<T> chunk, ByteBuffer nioBuffer, long handle, PooledByteBuf<T> buf, int reqCapacity) {
    // handle 为指向连续内存的一块指针
    chunk.initBufWithSubpage(buf, nioBuffer, handle, reqCapacity);
}
```

### PoolChunk#initBufWithSubpage

- 获取 Subpage 对象;
- [buf.init]()，初始化 SubPage 内存块到 PooledByteBuf 中。

```java
    void initBufWithSubpage(PooledByteBuf<T> buf, ByteBuffer nioBuffer, long handle, int reqCapacity) {
        initBufWithSubpage(buf, nioBuffer, handle, bitmapIdx(handle), reqCapacity);
    }

    private void initBufWithSubpage(PooledByteBuf<T> buf, ByteBuffer nioBuffer,
                                    long handle, int bitmapIdx, int reqCapacity) {
        assert bitmapIdx != 0;

        int memoryMapIdx = memoryMapIdx(handle);
        // 获取 Subpage 对象
        PoolSubpage<T> subpage = subpages[subpageIdx(memoryMapIdx)];
        assert subpage.doNotDestroy;
        assert reqCapacity <= subpage.elemSize;
        // 初始化 SubPage 内存块到 PooledByteBuf 中
        buf.init(
            this, nioBuffer, handle,
            runOffset(memoryMapIdx) + (bitmapIdx & 0x3FFFFFFF) * subpage.elemSize + offset,
                reqCapacity, subpage.elemSize, arena.parent.threadCache());
    }

    private int subpageIdx(int memoryMapIdx) {
        return memoryMapIdx ^ maxSubpageAllocs; // remove highest set bit, to get offset
    }
```
