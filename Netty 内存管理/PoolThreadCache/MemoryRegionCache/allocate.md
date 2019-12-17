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
