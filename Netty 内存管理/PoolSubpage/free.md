### free
　　
- 设置该 Subpage 对应的 bitmapIdx 为未分配，并将其赋值给 nextAvail，这样在 [getNextAvail]() 方法中可直接获取，不用遍历 bitmap 数组查找；
- 可用 Subpage 加一，大于 0，则添加 PoolSubpage 到链表中；
- 判断有没 Subpage 在使用，没则从链表中进行移除。

```java
    boolean free(PoolSubpage<T> head, int bitmapIdx) {
        if (elemSize == 0) {
            return true;
        }
        // 该 Subpage 在 bitmap 中的哪个 bits
        int q = bitmapIdx >>> 6;
        // 该 Subpage 在某个 bits 中的位置
        int r = bitmapIdx & 63;
        assert (bitmap[q] >>> r & 1) != 0;
        // 设置该 Subpage 为已分配
        bitmap[q] ^= 1L << r;
        // 设置已释放的 bitmapIdx 为可用的 nextAvail，这样在 [getNextAvail]()
        // 方法中可直接获取，不用遍历 bitmap 数组查找
        setNextAvail(bitmapIdx);
        // 可用 Subpage 加一，为 0，则添加 PoolSubpage 到链表中
        if (numAvail ++ == 0) {
            addToPool(head);
            return true;
        }

        // 有 Subpage 在使用
        if (numAvail != maxNumElems) {
            return true;
        } else {
            // 没有 Subpage 在使用，双向链表中，只有一个节点，不移除
            if (prev == next) {
                return true;
            }

            // 双向链表中有多个节点，则标记该节点已销毁，并从链表中移除
            doNotDestroy = false;
            removeFromPool();
            return false;
        }
    }
```

#### setNextAvail
　　设置已释放的 bitmapIdx 为可用的 nextAvail，这样在 [getNextAvail]() 方法中可直接获取，不用遍历 bitmap 数组查找。

```java
    private void setNextAvail(int bitmapIdx) {
        nextAvail = bitmapIdx;
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

#### removeFromPool
　　从双向链表中移除节点。

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