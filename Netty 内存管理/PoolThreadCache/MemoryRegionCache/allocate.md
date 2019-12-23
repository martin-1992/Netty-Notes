### allocate
　　如果 tinySubPageDirectCaches / smallSubPageDirectCaches / normalDirectCaches 中存在有缓存的对象，则从该队列中获取缓存的内存块，初始化到 PooledByteBuf 对象中，并返回是否分配成功。

- 从队列里弹出一个 Entry 对象（缓存的内存块）；
- initBuf，抽象方法，由子类 SubPageMemoryRegionCache 和 NormalMemoryRegionCache 来实现。以 SubPageMemoryRegionCache 为例，使用 ByteBuf 的变量保存传进来的参数，包括缓存的内存块地址 handle、chunk、偏移量 offset 等；
- 初始化完成后，对 Entry 进行回收，放回到（缓存的）对象池中。

```java
public final boolean allocate(PooledByteBuf<T> buf, int reqCapacity) {
    // 从队列里弹出一个 Entry 对象，下面代码用完后会进行回收
    Entry<T> entry = queue.poll();
    if (entry == null) {
        return false;
    }
    // 抽象方法，由子类 SubPageMemoryRegionCache 和 NormalMemoryRegionCache 来实现
    initBuf(entry.chunk, entry.nioBuffer, entry.handle, buf, reqCapacity);
    // 对 Entry（内存块） 进行回收，放回到（缓存的）对象池中
    entry.recycle();

    ++ allocations;
    return true;
}

protected abstract void initBuf(PoolChunk<T> chunk, ByteBuffer nioBuffer, long handle,
                                PooledByteBuf<T> buf, int reqCapacity);
```

### PoolThreadCache#SubPageMemoryRegionCache.initBuf
　　通过 chunk 进行分配，chunk 和 handle 可确认一块内存。以 SubPageMemoryRegionCache 为例，继续调用 PoolChunk#initBufWithSubpage 的方法。

```java
@Override
protected void initBuf(
        PoolChunk<T> chunk, ByteBuffer nioBuffer, long handle, PooledByteBuf<T> buf, int reqCapacity) {
    // handle 为指向连续内存的一块指针，通过 chunk 分配
    chunk.initBufWithSubpage(buf, nioBuffer, handle, reqCapacity);
}
```

### PoolChunk#initBufWithSubpage

- 在该 chunk 上，通过 handle 获取该对象的内存位置，获取 Subpage 对象（为 [PoolSubpage 数组](https://github.com/martin-1992/Netty-Notes/tree/master/Netty%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/PoolSubpage)，每个 PoolSubpage 为 Subpage 的分配情况表）;
- [buf.init](https://github.com/martin-1992/Netty-Notes/tree/master/Netty%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/PooledByteBuf)，将缓存的内存块地址 handle、chunk、偏移量 offset 等保存到 PooledByteBuf 中。

```java
    void initBufWithSubpage(PooledByteBuf<T> buf, ByteBuffer nioBuffer, long handle, int reqCapacity) {
        initBufWithSubpage(buf, nioBuffer, handle, bitmapIdx(handle), reqCapacity);
    }
    
    private final PoolSubpage<T>[] subpages;
    
    private void initBufWithSubpage(PooledByteBuf<T> buf, ByteBuffer nioBuffer,
                                    long handle, int bitmapIdx, int reqCapacity) {
        assert bitmapIdx != 0;
        // 通过 handle 获取该对象的内存位置
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

#### runOffset
　　计算偏移量。
  
```java
    private int runOffset(int id) {
        int shift = id ^ 1 << depth(id);
        return shift * runLength(id);
    }
 ```   
