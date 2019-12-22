### init

- 初始化 bitmap，根据 Subpage 大小来定义 bitmap 数组的长度 bitmapLength。bitmap 数组由 long 对象组成，一个 long 对象为 64 位，可记录 64 个 Subpage 是否已分配；
    1. 假设 Subpage 大小为 32B，则一个 Page（8K） 可划分 256 个 Subpage，其 bitmap 长度 bitmapLength 为 256 >> 6 = 4，使用 4 个 long 对象来记录 Subpage 的分配情况。
- 将创建的节点 PoolSubpage 添加到链表中。

```java
    void init(PoolSubpage<T> head, int elemSize) {
        doNotDestroy = true;
        // Subpage 的内存大小
        this.elemSize = elemSize;
        if (elemSize != 0) {
            // 根据 Subpage，将 page 均分为多个 Subpage，比如为 16B，则一个
            // page 可创建 512 个。32B，则可创建 256 个
            maxNumElems = numAvail = pageSize / elemSize;
            // 从 0 开始进行分配
            nextAvail = 0;
            // 除以 64，因为一个 long 对象为 64 位，可记录 64 个 Subpage 是否已分配
            bitmapLength = maxNumElems >>> 6;
            // 没有整除，加上 1
            if ((maxNumElems & 63) != 0) {
                bitmapLength ++;
            }
            // 初始化 bitmap
            for (int i = 0; i < bitmapLength; i ++) {
                bitmap[i] = 0;
            }
        }
        // 添加到链表中
        addToPool(head);
    }
```

#### addToPool
　　当前节点，插入到 head 节点后面。

```java
    private void addToPool(PoolSubpage<T> head) {
        assert prev == null && next == null;
        prev = head;
        next = head.next;
        next.prev = this;
        head.next = this;
    }
```