### allocateRun

- 计算在哪层二叉树进行内存分配；
- 获取二叉树中的节点值，用数组中的元素表示；
- 小于 0，表示没有获取到节点，返回 -1；
- 可用字节数减少，freeBytes 最大可用内存值为 16777216（16M，1 << 24）。
    1. runLength(id)，假设 id 为 4092，则 depth(id) = 11。前面提到层级值为 11，对应内存值为 8K，1 << log2ChunkSize - depth(id) =  1 << (24 - 11) = 8192，为 8K；
    2. freeBytes 为可用字节数，减去 8192 的字节值。

```java
    private long allocateRun(int normCapacity) {
        // 计算在哪层二叉树进行内存分配，第 11 层能分配 8K 的内存，第 10 层能分配 16K 的内存，以此类推。假设
        // normCapacity 为 31K，则 11 - (log2(32k) - 13) = 9，即在第 9 层分配 32 K，如果 normCapacity 大于
        // 16M，则 log2(normCapacity) 会大于 24，获取的 d 为负数
        int d = maxOrder - (log2(normCapacity) - pageShifts);
        // 获取节点
        int id = allocateNode(d);
        // 小于 0，表示没有获取到节点，返回 -1
        if (id < 0) {
            return id;
        }
        // 可用字节数减少
        freeBytes -= runLength(id);
        return id;
    }
    
    // 默认为 log2(16M) = 24
    private final int log2ChunkSize;
    
    /**
     * 假设 id 为 4092，则 depth(id) = 11，前面提到层级值为 11，对应内存值为 8K，
     * 1 << log2ChunkSize - depth(id) =  1 << (24 - 11) = 8192，为 8K
     **/
    private int runLength(int id) {
        return 1 << log2ChunkSize - depth(id);
    }
    
    /**
     * 该节点对应的层级值
     **/
    private byte depth(int id) {
        return depthMap[id];
    }
```
