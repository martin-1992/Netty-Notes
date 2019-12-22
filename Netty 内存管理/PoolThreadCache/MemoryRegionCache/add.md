### add
　　添加 Entry（缓存）到队列中。

```java
    @SuppressWarnings("unchecked")
    public final boolean add(PoolChunk<T> chunk, ByteBuffer nioBuffer, long handle) {
        Entry<T> entry = newEntry(chunk, nioBuffer, handle);
        // 添加到队列
        boolean queued = queue.offer(entry);
        // 添加失败，表示队列已满，则回收 Entry 对象
        if (!queued) {
            // If it was not possible to cache the chunk, immediately recycle the entry
            entry.recycle();
        }

        return queued;
    }
```
