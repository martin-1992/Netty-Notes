### allocateSubpage

- 调用 findSubpagePoolHead，按字节选取 PoolSubpage，tinySubpagePools 和 smallSubpagePools 数组中的每个值为链表，所以获取的是头节点 PoolSubpage；
- 调用 [allocateNode]()，获取 bitmap 的节点值。如果节点为 -1，表示没有内存可分配，返回；
- 剩余可用字节数 freeBytes 减少一个 page 大小；
- 根据节点值，获得对应的 Subpage 数组（0~2047）的编号，通过编号获取 PoolSubpage 对象；
    1. 如果 [PoolSubpage]() 为空，则创建；
    2. 不为空，则进行初始化 [subpage.init]()。
- [subpage.allocate()]()，分配 PoolSubpage 内存块。

```java
    // PoolSubpage 数组
    private final PoolSubpage<T>[] subpages;

    private long allocateSubpage(int normCapacity) {
        // 按字节选取 PoolSubpage，tinySubpagePools 和 smallSubpagePools 数组中的每个值
        // 都是用链表形式的，所以获取的是头节点 PoolSubpage
        PoolSubpage<T> head = arena.findSubpagePoolHead(normCapacity);
        // 默认 11，二叉树的高度值，Subpage 使用的是最底层，一个为 8K
        int d = maxOrder;
        // 使用同步，因为会修改链表结构，线程安全
        synchronized (head) {
            // 获取节点
            int id = allocateNode(d);
            // 节点为 -1，没有内存可分配，返回
            if (id < 0) {
                return id;
            }

            final PoolSubpage<T>[] subpages = this.subpages;
            // Page 大小，默认 8KB = 8192B
            final int pageSize = this.pageSize;
            // 剩余可用字节数减少
            freeBytes -= pageSize;

            // 获得节点对应的 Subpage 数组（0~2047）的编号
            int subpageIdx = subpageIdx(id);
            // 获取 PoolSubpage 对象
            PoolSubpage<T> subpage = subpages[subpageIdx];
            if (subpage == null) {
                // 为空，则创建 PoolSubpage
                subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity);
                subpages[subpageIdx] = subpage;
            } else {
                // 不为空，则重新初始化 PoolSubpage
                subpage.init(head, normCapacity);
            }
            // 分配 PoolSubpage 内存块
            return subpage.allocate();
        }
    }
```

### findSubpagePoolHead
　　按字节选取 PoolSubpage，tinySubpagePools 和 smallSubpagePools 数组中的每个值都是用链表形式的，所以获取的是头节点 PoolSubpage。

```java
    PoolSubpage<T> findSubpagePoolHead(int elemSize) {
        int tableIdx;
        PoolSubpage<T>[] table;
        // 小于 512 字节，使用 tinySubpagePools
        if (isTiny(elemSize)) { // < 512
            // 根据字节 elemSize 来选取哪个 PoolSubpage，tinySubpagePools 为 PoolSubpage 数组，按
            // [null, 16 字节的 PoolSubpage, 32 字节的 PoolSubpage，...] 分配
            tableIdx = elemSize >>> 4;
            table = tinySubpagePools;
        } else {
            // 512 到 8K 的，根据字节 elemSize 来选取哪个 PoolSubpage，smallSubpagePools 为 PoolSubpage 数组，按
            // [512, 1024 字节的 PoolSubpage, 2048 字节的 PoolSubpage，4096 字节的 PoolSubpage] 分配
            tableIdx = 0;
            elemSize >>>= 10;
            while (elemSize != 0) {
                elemSize >>>= 1;
                tableIdx ++;
            }
            table = smallSubpagePools;
        }

        return table[tableIdx];
    }
```

#### subpageIdx

```java
    private int subpageIdx(int memoryMapIdx) {
        return memoryMapIdx ^ maxSubpageAllocs; // remove highest set bit, to get offset
    }
```