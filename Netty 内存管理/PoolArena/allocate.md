### allocate

- 从对象池 Recycler 获取 ByteBuf 对象，如果没有，则直接创建；
- 初始化 ByteBuf 对象，即重置读写指针、重置读写指针的标记值等；
- 在该 ByteBuf 对象上，进行内存分配。

```java
    PooledByteBuf<T> allocate(PoolThreadCache cache, int reqCapacity, int maxCapacity) {
        PooledByteBuf<T> buf = newByteBuf(maxCapacity);
        // 内存分配
        allocate(cache, buf, reqCapacity);
        return buf;
    }
```


### PoolArena#newByteBuf
　　默认情况下，是有 Unsafe 的，即调用 PooledUnsafeDirectByteBuf。

```java
        @Override
        protected PooledByteBuf<ByteBuffer> newByteBuf(int maxCapacity) {
            // 默认情况下，是有 Unsafe 的
            if (HAS_UNSAFE) {
                return PooledUnsafeDirectByteBuf.newInstance(maxCapacity);
            } else {
                return PooledDirectByteBuf.newInstance(maxCapacity);
            }
        }
```


#### PooledUnsafeDirectByteBuf#newInstance
　　
- [Recycler#get]()，获取 ByteBuf 对象，如果没有，则直接创建；
- [PooledByteBuf#reuse]()，初始化 ByteBuf 对象，即重置读写指针、重置读写指针的标记值等。

```java
    // ByteBuf 的对象池
    private static final Recycler<PooledUnsafeDirectByteBuf> RECYCLER = new Recycler<PooledUnsafeDirectByteBuf>() {
        @Override
        protected PooledUnsafeDirectByteBuf newObject(Handle<PooledUnsafeDirectByteBuf> handle) {
            // 如果没对应的 ByteBuf 对象，则直接创建，对象是绑定在 handle.value 上，
            // 由 handle 调用 recycle 方法来回收对象
            return new PooledUnsafeDirectByteBuf(handle, 0);
        }
    };

    static PooledUnsafeDirectByteBuf newInstance(int maxCapacity) {
        // 获取 ByteBuf 对象，如果没有，则直接创建
        PooledUnsafeDirectByteBuf buf = RECYCLER.get();
        // 可能是回收站里拿出来的，进行复用，设置读写指针、最大扩容容量等
        buf.reuse(maxCapacity);
        // 获取初始化好的 ByteBuf
        return buf;
    }
    
    /**
     * PooledByteBuf#reuse
     */
    final void reuse(int maxCapacity) {
        // 最大扩容容量
        maxCapacity(maxCapacity);
        // 设置应用，表示当前的 ByteBuf 被多个地方进行引用
        setRefCnt(1);
        // 重置读写指针
        setIndex0(0, 0);
        // 重置读写指针的标记值
        discardMarks();
    }
```


### PoolArena#allocate

- 标准化请求的内存容量，为 2 的次方，从大到小来判断；
- 缓存上进行内存分配；
- 内存堆上进行内存分配。

```java
    private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
        // 标准化请求的内存容量，为 2 的次方，从大到小来判断
        final int normCapacity = normalizeCapacity(reqCapacity);
        // 小于 pageSize，即为 tiny 或 small，capacity < pageSize
        if (isTinyOrSmall(normCapacity)) {
            int tableIdx;
            PoolSubpage<T>[] table;
            // 使用位运算，小于 512 是 tiny，而 512 到 pageSize 则为 small
            boolean tiny = isTiny(normCapacity);
            if (tiny) { // < 512
                // allocateTiny、allocateSmall、allocateNormal 处理逻辑都类似

                if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = tinyIdx(normCapacity);
                table = tinySubpagePools;
            } else {
                if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = smallIdx(normCapacity);
                table = smallSubpagePools;
            }

            final PoolSubpage<T> head = table[tableIdx];

            /**
             * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
             * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
             */
            synchronized (head) {
                final PoolSubpage<T> s = head.next;
                if (s != head) {
                    assert s.doNotDestroy && s.elemSize == normCapacity;
                    long handle = s.allocate();
                    assert handle >= 0;
                    s.chunk.initBufWithSubpage(buf, null, handle, reqCapacity);
                    incTinySmallAllocation(tiny);
                    return;
                }
            }
            synchronized (this) {
                allocateNormal(buf, reqCapacity, normCapacity);
            }

            incTinySmallAllocation(tiny);
            return;
        }
        // 小于 16M 时的缓存分配，以 page 来分的
        if (normCapacity <= chunkSize) {
            // 在缓存上进行内存分配，分配成功则 return
            if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            // 如果在缓存上分配不成功，则在做实际的内存分配，首次是分配不到，因为没有缓存
            synchronized (this) {
                // page 级别的分配
                allocateNormal(buf, reqCapacity, normCapacity);
                ++allocationsNormal;
            }
        } else {
            // 如果分配的内存 normCapacity 大于 chunkSize，则使用 allocateHuge 在内存上分配
            // Huge allocations are never served via the cache so just call allocateHuge
            allocateHuge(buf, reqCapacity);
        }
    }
```

