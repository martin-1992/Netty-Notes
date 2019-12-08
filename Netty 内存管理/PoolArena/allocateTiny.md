### PoolThreadCache#allocateTiny

- cacheForTiny，找到 MemoryRegionCache 的节点；
- allocate，分配缓存。

```java
    boolean allocateTiny(PoolArena<?> area, PooledByteBuf<?> buf, int reqCapacity, int normCapacity) {
        // cacheForTiny，找到 MemoryRegionCache 的节点
        return allocate(cacheForTiny(area, normCapacity), buf, reqCapacity);
    }
```


### PoolThreadCache#cacheForTiny
　　根据请求容量获取分配在内存数组中的位置，比如请求容量 normCapacity 为 32，则 32 >>> 4 为 2，分配在 tiny 数组中索引位置为 2。

```java
    private MemoryRegionCache<?> cacheForTiny(PoolArena<?> area, int normCapacity) {
        // 根据请求容量获取分配在内存数组中的位置，比如请求容量 normCapacity 为 32，
        // 则 32 >>> 4 为 2，分配在 tiny 数组中索引位置为 2
        int idx = PoolArena.tinyIdx(normCapacity);
        // 分配内存的数组为直接内存
        if (area.isDirect()) {
            return cache(tinySubPageDirectCaches, idx);
        }
        // 堆内内存 tiny 数组
        return cache(tinySubPageHeapCaches, idx);
    }
```


#### PoolArena#tinyIdx
　　把 normCapacity 除以 16，因为 tiny 的数组是按 16 的倍数排的，比如 tiny[1]=16B，tiny[2]=32B，tiny[3]=48B，这样如果是 32B，则 32 除以 16，取第二个。

```java
    static int tinyIdx(int normCapacity) {
        return normCapacity >>> 4;
    }
```


#### PoolThreadCache#cacheForTiny

-

```java
    private static <T> MemoryRegionCache<T> cache(MemoryRegionCache<T>[] cache, int idx) {
        // 参数校验
        if (cache == null || idx > cache.length - 1) {
            return null;
        }
        // 使用下标，获取数组的值，比如 32B / 16 = 2，获取 cache[2]，即为 32B
        return cache[idx];
    }
```


### allocate

```java
    private boolean allocate(MemoryRegionCache<?> cache, PooledByteBuf buf, int reqCapacity) {
        if (cache == null) {
            return false;
        }
        // 获取到节点 MemoryRegionCache，需要从该节点的 queue 里弹出一个 entry 给 ByteBuf
        boolean allocated = cache.allocate(buf, reqCapacity);
        if (++ allocations >= freeSweepAllocationThreshold) {
            allocations = 0;
            trim();
        }
        return allocated;
    }
```

