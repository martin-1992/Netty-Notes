### allocate
　　分配内存，判断请求容量是否大于等于 pagSize，是则分配 page，否则分配 subPage。

```java
    // 判断分配请求内存是否为 Tiny/Small，即分配 Subpage 内存块
    private final int subpageOverflowMask;

    boolean allocate(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
        final long handle;
        // 判断请求容量是否大于等于 pagSize，是则分配 page，否则分配 subPage
        if ((normCapacity & subpageOverflowMask) != 0) {
            handle =  allocateRun(normCapacity);
        } else {
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