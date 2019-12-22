### trim
　　allocations 为分配次数，每调用一次 allocate 方法时，allocations 加一。假设队列为 tiny，则队列大小 size 为 512，当 allocations 分配次数小于 512 时，则将还没分配的内存空间释放，防止内存泄漏。

```java
    public final void trim() {
        // allocations 表示已经分配出去的 ByteBuf 个数
        int free = size - allocations;
        allocations = 0;

        // We not even allocated all the number that are
        // 在一定阈值内还没被分配出去的空间将被释放
        if (free > 0) {
            // 释放队列中的节点
            free(free);
        }
    }
```

#### free
　　将队列中 Entry 对象清空。

```java
        public final int free() {
            return free(Integer.MAX_VALUE);
        }

        private int free(int max) {
            int numFreed = 0;
            for (; numFreed < max; numFreed++) {
                Entry<T> entry = queue.poll();
                if (entry != null) {
                    freeEntry(entry);
                } else {
                    // all cleared
                    return numFreed;
                }
            }
            return numFreed;
        }
```

#### freeEntry
　　回收 Entry 、handle 对象，释放缓存的内存块回 Chunk 中。

```java
    @SuppressWarnings({ "unchecked", "rawtypes" })
    private  void freeEntry(Entry entry) {
        PoolChunk chunk = entry.chunk;
        long handle = entry.handle;
        ByteBuffer nioBuffer = entry.nioBuffer;

        // recycle now so PoolChunk can be GC'ed.
        entry.recycle();

        chunk.arena.freeChunk(chunk, handle, sizeClass, nioBuffer);
    }
```
