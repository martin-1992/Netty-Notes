### PooledByteBufAllocator
　　基于内存池的 ByteBuf 的分配器。内存池，不像对象池只有单一对象（只需创建固定大小的对象），由于存在不同内存大小的消息，需要创建不同的 ByteBuf 来分配，同时达到复用。所以会创建了三种通用的、可复用的、不同规格大小的连续内存。

- defaultPageSize，默认 page 大小为 8k（8192）；
- defaultMaxOrder，二叉树高度，默认为 11。8192 << 11 为一个 16M 的 chunk，Netty 中可复用的最大一段连续内存。即 2048（2^11）个 8k 的page 组成一个 16M 的 chunk；
- DEFAULT_TINY_CACHE_SIZE，tiny 大小的 ByteBuf 队列（PoolThreadCache#MemoryRegionCache 创建的 MpscArrayQueue），默认长度为 512，可写入 0 ~ 512B 的数据；
- DEFAULT_SMALL_CACHE_SIZE，small 大小的 ByteBuf 队列，默认长度为 256，可写入 512B ~ 8K 的数据；
- DEFAULT_NORMAL_CACHE_SIZE，normal 大小的 ByteBuf 队列，默认长度为 64，可写入 8K ~ 16M 的数据；

```java
    private static final int DEFAULT_PAGE_SIZE;
    private static final int DEFAULT_MAX_ORDER; // 8192 << 11 = 16 MiB per chunk

    private static final int MIN_PAGE_SIZE = 4096;
    private static final int MAX_CHUNK_SIZE = (int) (((long) Integer.MAX_VALUE + 1) / 2);

    static {
        // page 大小，默认 8 k
        int defaultPageSize = SystemPropertyUtil.getInt("io.netty.allocator.pageSize", 8192);
        Throwable pageSizeFallbackCause = null;
        // 参数校验，省略...
        
        DEFAULT_PAGE_SIZE = defaultPageSize;
        // 二叉树高度，默认 11
        int defaultMaxOrder = SystemPropertyUtil.getInt("io.netty.allocator.maxOrder", 11);
        Throwable maxOrderFallbackCause = null;
        // 参数校验，省略...
        
        DEFAULT_MAX_ORDER = defaultMaxOrder;

        // Determine reasonable default for nHeapArena and nDirectArena.
        // Assuming each arena has 3 chunks, the pool should not consume more than 50% of max memory.
        final Runtime runtime = Runtime.getRuntime();

        // 一个 CPU 等于两个的 arena，即创建 2 倍 CPU 核数的 arena，之所以是 2，是因为能使用位运算，加快速度
        final int defaultMinNumArena = NettyRuntime.availableProcessors() * 2;
        // 每个 chunk 大小默认 16M，即 8192 << 11 = 16M
        final int defaultChunkSize = DEFAULT_PAGE_SIZE << DEFAULT_MAX_ORDER;
        DEFAULT_NUM_HEAP_ARENA = Math.max(0,
                SystemPropertyUtil.getInt(
                        "io.netty.allocator.numHeapArenas",
                        (int) Math.min(
                                defaultMinNumArena,
                                runtime.maxMemory() / defaultChunkSize / 2 / 3)));
        DEFAULT_NUM_DIRECT_ARENA = Math.max(0,
                SystemPropertyUtil.getInt(
                        "io.netty.allocator.numDirectArenas",
                        (int) Math.min(
                                defaultMinNumArena,
                                PlatformDependent.maxDirectMemory() / defaultChunkSize / 2 / 3)));

        // tiny 大小的 ByteBuf 队列，默认长度为 512
        DEFAULT_TINY_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.tinyCacheSize", 512);
        // small 大小的 ByteBuf 队列，默认长度为 256
        DEFAULT_SMALL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.smallCacheSize", 256);
        // normal 大小的 ByteBuf 队列，默认长度为 64
        DEFAULT_NORMAL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.normalCacheSize", 64);

        // ...
        }
    }
```

### PoolThreadLocalCache#initialValue

- 创建 heapArena 和 directArena，为内存分配器（Netty 的内存分配逻辑通过 Arena 实现的）。heapArena 和 directArena 是线程绑定的，即每个线程都有各自的 heapArena 和 directArena，保证线程安全；
- 判断当前线程是否满足条件，useCacheForAllThreads 为 true 或为 FastThreadLocalThread 的实例，即线程安全；
- 调用构造函数 [PoolThreadCache]()，把 heapArena、directArena、tinyCacheSize、smallCacheSize 和 normalCacheSize 当做变量传递进去，绑定当前线程实例。

```java
    private final PoolArena<byte[]>[] heapArenas;
    private final PoolArena<ByteBuffer>[] directArenas;

    final class PoolThreadLocalCache extends FastThreadLocal<PoolThreadCache> {
        private final boolean useCacheForAllThreads;

        PoolThreadLocalCache(boolean useCacheForAllThreads) {
            this.useCacheForAllThreads = useCacheForAllThreads;
        }

        /**
         * 每个线程都有唯一的 PoolThreadCache
         * @return
         */
        @Override
        protected synchronized PoolThreadCache initialValue() {
            // 创建 heapArena 和 directArena
            final PoolArena<byte[]> heapArena = leastUsedArena(heapArenas);
            final PoolArena<ByteBuffer> directArena = leastUsedArena(directArenas);
            // 获取当前线程，用于判定是否为 FastThreadLocalThread
            Thread current = Thread.currentThread();
            if (useCacheForAllThreads || current instanceof FastThreadLocalThread) {
                // 调用构造函数创建 PoolThreadCache
                return new PoolThreadCache(
                        // 把 heapArena、directArena、tinyCacheSize、smallCacheSize 和
                        // normalCacheSize 当做变量传递进去，绑定当前线程
                        heapArena, directArena, tinyCacheSize, smallCacheSize, normalCacheSize,
                        DEFAULT_MAX_CACHED_BUFFER_CAPACITY, DEFAULT_CACHE_TRIM_INTERVAL);
            }
            // No caching so just use 0 as sizes.
            return new PoolThreadCache(heapArena, directArena, 0, 0, 0, 0, 0);
        }

        // ...
    }
```

### newDirectBuffer

- threadCache.get()，因为调用 newDirectBuffer 可能是多线程的，所以会使用类似 ThreadLocal 的机制，即 PoolThreadLocalCache 来获取线程局部缓存；
- cache.directArena，从线程局部缓存中获取当前线程的 PoolArena。在前面提到 PoolThreadLocalCache#initialValue 会创建 PoolArena；
- directArena.allocate，内存分配。

```java
    private final PoolThreadLocalCache threadCache;

    @Override
    protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        PoolThreadCache cache = threadCache.get();
        PoolArena<ByteBuffer> directArena = cache.directArena;

        final ByteBuf buf;
        if (directArena != null) {
            buf = directArena.allocate(cache, initialCapacity, maxCapacity);
        } else {
            buf = PlatformDependent.hasUnsafe() ?
                    UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity) :
                    new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
        }

        return toLeakAwareBuffer(buf);
    }
```
