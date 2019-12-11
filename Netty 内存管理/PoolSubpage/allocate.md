### allocate
　　分配 Subpage 内存块。

- 判断，如果剩余可用的 Subpage 数为 0，则直接返回；
- 获取可用的 Subpage 的索引位置，将该 Subpage 在 bitmap 中设置为已分配；
    1. 根据 bitmap 的长度 bitmapLength，遍历每个 bits（long 类型）；
    2. 对 bits 取反并判断是否为 0，如果为 0，表示已全部分配完，遍历下个 bits。不为 0，则获取该 bits；
    3. 对该 bits（64 位） 的每一位进行遍历，获取某一位不为 1 的，返回该位在 bitmap 中的位置。
- 如果剩余可用的 Subpage 为 0，则将 PoolSubpage 从链表中移除。

```java
    // 剩余可用 Subpage 的数量
    private int numAvail;

    // 是否没有销毁
    boolean doNotDestroy;

    long allocate() {
        if (elemSize == 0) {
            return toHandle(0);
        }

        // 剩余可用的 subpage 数为 0，则直接返回
        if (numAvail == 0 || !doNotDestroy) {
            return -1;
        }
        // 获取可用的 Subpage，在 bits 中的位置（0~511）
        final int bitmapIdx = getNextAvail();
        // 获取该可用的 Subpage 在哪个 bits 中，比如 Subpage 为 135，则在第 2 个 bits 中
        int q = bitmapIdx >>> 6;
        // 获取该可用的 Subpage 在 bits 中的位置，比如 Subpage 为 135，则在第 2 个 bits 中
        // 的第 7 个索引位置
        int r = bitmapIdx & 63;
        assert (bitmap[q] >>> r & 1) == 0;
        // 设置该 Subpage 为已分配
        bitmap[q] |= 1L << r;

        // 可用 Subpage 数减一
        if (-- numAvail == 0) {
            // 如果剩余可用 Subpage 为 0，则从链表中移除
            removeFromPool();
        }
        // 计算 handle 值
        return toHandle(bitmapIdx);
    }
```

#### getNextAvail
　　获取下一个可用的 Subpage 在 bigmap 的位置。

- 存在可用的 nextAvail，则直接返回；
- 调用 findNextAvail，查找可用的 findNextAvail。

```java
    private int getNextAvail() {
        int nextAvail = this.nextAvail;
        // 存在下一个可用位置，获取返回，并将 nextAvail 置为 -1，下次需要寻找
        if (nextAvail >= 0) {
            this.nextAvail = -1;
            return nextAvail;
        }
        return findNextAvail();
    }
```

#### findNextAvail
　　遍历 bitmap，如果某个某个 bits（long 类型），按位取反不为 0，则存在可分配的 Subpage，调用 findNextAvail0，从该 bits 查找可分配的 Subpage 所在的索引。

```java
    /**
     * 获取 nextAvail
     **/
    private int findNextAvail() {
        // 当前 bitmap 数组
        final long[] bitmap = this.bitmap;
        // bitmap 的长度，如果为 32B，则为 4，16B 为 8
        final int bitmapLength = this.bitmapLength;
        // 遍历 bitmap
        for (int i = 0; i < bitmapLength; i ++) {
            long bits = bitmap[i];
            // 按位取反，如果该 long 对象 bits 全为 1，取反则为 0，即没有可分配的
            // Subpage，如果不为 0，则存在可分配的 Subpage
            if (~bits != 0) {
                // 从该 bits 查找可用的 nextAvail
                return findNextAvail0(i, bits);
            }
        }
        return -1;
    }
```

#### findNextAvail0
　　获取可用的 Subpage，在 bits 中的位置（0~511，如果 bitmapLength 为 8 的话。为 4，则是 0~255）。

```java
    // Subpage 的总数
    private int maxNumElems;

    private int findNextAvail0(int i, long bits) {
        // Subpage 总数
        final int maxNumElems = this.maxNumElems;
        // 乘以 64，一个 long 对象可记录 64 个 Subpage 的分配情况
        final int baseVal = i << 6;
        // 遍历该 bits（long 类型），通过右移 >>> 来遍历下一位
        for (int j = 0; j < 64; j ++) {
            // bits 为 0，则未分配
            if ((bits & 1) == 0) {
                // 检查该值是否存在于 bitmap 中，即判断要找的值是否超过 Subpage 总数
                int val = baseVal | j;
                // 不超过 Subpage 总数，返回当前值
                if (val < maxNumElems) {
                    return val;
                } else {
                    break;
                }
            }
            // 右移 bits，获取下一位
            bits >>>= 1;
        }
        return -1;
    }
```

#### removeFromPool
　　移除节点。

```java
    private void removeFromPool() {
        assert prev != null && next != null;
        // 前后
        prev.next = next;
        next.prev = prev;
        next = null;
        prev = null;
    }
```