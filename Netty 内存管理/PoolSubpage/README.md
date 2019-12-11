### PoolSubpage
　　bitmap 为 long 对象的数组，是 Subpage 的分配情况表。long 对象为 64 位，可记录 64 个 Subpage 是否分配，分配则该 long 对象对应的某位为 1，没分配则为 0。<br />
　　假设 Subpage 为 16B，一个 Page 默认大小为 8K，可分配 8 * 1024 / 16 = 512 个 Subpage，bitmap 数组大小为 512 / 64 = 8。如果 Subpage 为 32B，则可分配 256 个 Subpage，bitmap 数组大小为 256 / 64 = 4。

```java
final class PoolSubpage<T> implements PoolSubpageMetric {

    // 所属 PoolChunk 对象
    final PoolChunk<T> chunk;
    // 在 memoryMap 中对应的节点索引
    private final int memoryMapIdx;
    // 偏移量
    private final int runOffset;
    // page 大小，默认为 8K
    private final int pageSize;

    // Subpage 的分配情况表
    private final long[] bitmap;
    
    // Subpage 链表
    PoolSubpage<T> prev;
    PoolSubpage<T> next;

    // 是否没有销毁
    boolean doNotDestroy;
    // Subpage 的内存大小，比如为 16B，则一个 page(8K) 可创建 512 个。32B，则可创建 256 个
    int elemSize;
    // Subpage 的总数
    private int maxNumElems;
    // bimap 的长度
    private int bitmapLength;
    // 下一个可分配 Subpage 的数组位置
    private int nextAvail;
    // 剩余可用 Subpage 的数量
    private int numAvail;
}
```

### 构造函数
　　调用 [init]() 方法

```java
    /**
     * 构造函数，创建头节点
     **/
    PoolSubpage(int pageSize) {
        chunk = null;
        memoryMapIdx = -1;
        runOffset = -1;
        elemSize = -1;
        this.pageSize = pageSize;
        bitmap = null;
    }

    /**
     * 构造函数，page 节点
     **/
    PoolSubpage(PoolSubpage<T> head, PoolChunk<T> chunk, int memoryMapIdx, int runOffset, int pageSize, int elemSize) {
        this.chunk = chunk;
        this.memoryMapIdx = memoryMapIdx;
        this.runOffset = runOffset;
        this.pageSize = pageSize;
        // 创建 bitmap 数组，page 大小默认为 8K，8192 >>> 10 = 8
        bitmap = new long[pageSize >>> 10];
        //
        init(head, elemSize);
    }
}
```