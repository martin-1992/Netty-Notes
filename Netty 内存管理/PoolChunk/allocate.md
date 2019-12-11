### allocate
　　分配内存，判断请求容量是否大于等于 pagSize，是则分配 page，否则分配 subPage。

```java
    private final Deque<ByteBuffer> cachedNioBuffers;

    // 判断分配请求内存是否为 Tiny/Small，即分配 Subpage 内存块
    private final int subpageOverflowMask;

    boolean allocate(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
        final long handle;
        // 判断请求容量是否大于等于 pagSize，是则分配 page，否则分配 subPage
        if ((normCapacity & subpageOverflowMask) != 0) {
            // 获取节点值
            handle =  allocateRun(normCapacity);
        } else {
            // 获取节点值
            handle = allocateSubpage(normCapacity);
        }

        if (handle < 0) {
            return false;
        }
        ByteBuffer nioBuffer = cachedNioBuffers != null ? cachedNioBuffers.pollLast() : null;
        initBuf(buf, nioBuffer, handle, reqCapacity);
        return true;
    }
```

### initBuf

- [buf.init]()，初始化 Page 内存块到 PooledByteBuf 中；
- initBufWithSubpage，初始化 SubPage 内存块到 PooledByteBuf 中。

```java
    void initBuf(PooledByteBuf<T> buf, ByteBuffer nioBuffer, long handle, int reqCapacity) {
        // 节点编号
        int memoryMapIdx = memoryMapIdx(handle);
        int bitmapIdx = bitmapIdx(handle);
        // Page 内存块
        if (bitmapIdx == 0) {
            // 该节点对应的层级值
            byte val = value(memoryMapIdx);
            // 判断该节点是否已分配，不可用了
            assert val == unusable : String.valueOf(val);
            // 初始化 Page 内存块到 PooledByteBuf 中
            buf.init(this, nioBuffer, handle, runOffset(memoryMapIdx) + offset,
                    reqCapacity, runLength(memoryMapIdx), arena.parent.threadCache());
        } else {
            // 初始化 SubPage 内存块到 PooledByteBuf 中
            initBufWithSubpage(buf, nioBuffer, handle, bitmapIdx, reqCapacity);
        }
    }
```

#### initBufWithSubpage

- 获取 Subpage 对象;
- [buf.init]()，初始化 SubPage 内存块到 PooledByteBuf 中。

```java
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
