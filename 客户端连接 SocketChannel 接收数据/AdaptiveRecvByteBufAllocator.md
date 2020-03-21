### AdaptiveRecvByteBufAllocator
　　自适应数据大小的分配器，根据这次接收数据的实际大小，调整接下来的 ByteBuf 大小。<br />
　　Netty 会使用一个容量表，0-496 为 16 递增，而 512-1073741824 为两倍递增，即容量表为 [16, 32, 48, ... , 496, 512, 1024, 2048, ... , 1073741824]。**通过二分法，从容量表中找到 ByteBuf 最适合的大小。**

```java
public class AdaptiveRecvByteBufAllocator extends DefaultMaxMessagesRecvByteBufAllocator {

    /**
     * 缩容操作最小容量，默认为 64
     */
    static final int DEFAULT_MINIMUM = 64;

    /**
     * 默认初始容量为 1024，之后如果太大，会缩小
     */
    static final int DEFAULT_INITIAL = 1024;

    /**
     * 扩容操作最大容量，默认为 65536
     */
    static final int DEFAULT_MAXIMUM = 65536;

    /**
     * 扩容步数
     */
    private static final int INDEX_INCREMENT = 4;

    /**
     * 缩容步数
     */
    private static final int INDEX_DECREMENT = 1;

    private static final int[] SIZE_TABLE;

    static {
        // 容量表，[16, 32, 48, ... , 496]
        List<Integer> sizeTable = new ArrayList<Integer>();
        for (int i = 16; i < 512; i += 16) {
            sizeTable.add(i);
        }

        // [512, 1024, 2048, ... , 1073741824]，从 512 开始进行两倍扩容，直到溢出转为负数终止
        for (int i = 512; i > 0; i <<= 1) {
            sizeTable.add(i);
        }

        // 将容量表的值复制到 SIZE_TABLE
        SIZE_TABLE = new int[sizeTable.size()];
        for (int i = 0; i < SIZE_TABLE.length; i ++) {
            SIZE_TABLE[i] = sizeTable.get(i);
        }
    }

    /**
     * @deprecated There is state for {@link #maxMessagesPerRead()} which is typically based upon channel type.
     */
    @Deprecated
    public static final AdaptiveRecvByteBufAllocator DEFAULT = new AdaptiveRecvByteBufAllocator();
    
    private final int minIndex;
    private final int maxIndex;
    private final int initial;

    public AdaptiveRecvByteBufAllocator() {
        this(DEFAULT_MINIMUM, DEFAULT_INITIAL, DEFAULT_MAXIMUM);
    }

    public AdaptiveRecvByteBufAllocator(int minimum, int initial, int maximum) {
        // 参数校验，是否大于 0
        checkPositive(minimum, "minimum");
        if (initial < minimum) {
            throw new IllegalArgumentException("initial: " + initial);
        }
        if (maximum < initial) {
            throw new IllegalArgumentException("maximum: " + maximum);
        }

        // 获取最小容量 minimum 在容量表中的索引位置
        int minIndex = getSizeTableIndex(minimum);
        // 容量表对应的容量小于最小容量，则取下一格，即比最小容量大一点
        if (SIZE_TABLE[minIndex] < minimum) {
            this.minIndex = minIndex + 1;
        } else {
            this.minIndex = minIndex;
        }

        // 同理，获取最大容量在容量表中的索引
        int maxIndex = getSizeTableIndex(maximum);
        if (SIZE_TABLE[maxIndex] > maximum) {
            this.maxIndex = maxIndex - 1;
        } else {
            this.maxIndex = maxIndex;
        }

        this.initial = initial;
    }

    @SuppressWarnings("deprecation")
    @Override
    public Handle newHandle() {
        return new HandleImpl(minIndex, maxIndex, initial);
    }

    @Override
    public AdaptiveRecvByteBufAllocator respectMaybeMoreData(boolean respectMaybeMoreData) {
        super.respectMaybeMoreData(respectMaybeMoreData);
        return this;
    }
}
```

### getSizeTableIndex
　　使用二分法，从容量表中找到 ByteBuf 最适合的大小。

```java
    private static int getSizeTableIndex(final int size) {
        for (int low = 0, high = SIZE_TABLE.length - 1;;) {
            if (high < low) {
                return low;
            }
            if (high == low) {
                return high;
            }

            // 二分法
            int mid = low + high >>> 1;
            // 左中位数
            int a = SIZE_TABLE[mid];
            // 右中位数
            int b = SIZE_TABLE[mid + 1];
            // 二分法判断
            if (size > b) {
                low = mid + 1;
            } else if (size < a) {
                high = mid - 1;
            } else if (size == a) {
                return mid;
            } else {
                return mid + 1;
            }
        }
    }
```

### HandleImpl
　　使用 record()，来对接收数据的 ByteBuf 进行扩容或缩容，根据实际大小来判断，防止空间浪费。

- 需要判断两次才进行缩容，即实际读取大小小于缩容大小，则进行缩容。缩容是让容量表对应的索引减一，返回容量表索引对应的大小；
- 扩容是一次判断就行，而且扩容是一次加 4，即容量表对应的索引加四。

```java
    private final class HandleImpl extends MaxMessageHandle {
        private final int minIndex;
        private final int maxIndex;
        private int index;
        private int nextReceiveBufferSize;
        private boolean decreaseNow;

        HandleImpl(int minIndex, int maxIndex, int initial) {
            this.minIndex = minIndex;
            this.maxIndex = maxIndex;

            // 根据初始容量获取最适合的 ByteBuf 大小的索引
            index = getSizeTableIndex(initial);
            nextReceiveBufferSize = SIZE_TABLE[index];
        }

        @Override
        public void lastBytesRead(int bytes) {
            if (bytes == attemptedBytesRead()) {
                // 看是否需要扩容或缩小内存
                record(bytes);
            }
            // 将 bytes 记录为上次读取的大小
            super.lastBytesRead(bytes);
        }

        @Override
        public int guess() {
            // 默认为 1024，根据情况在扩容或缩小
            return nextReceiveBufferSize;
        }

        /**
         * 对接收数据的 ByteBuf 进行扩容或缩容，根据实际大小来判断，防止空间浪费
         * @param actualReadBytes
         */
        private void record(int actualReadBytes) {
            // index - INDEX_DECREMENT - 1，表示缩容，INDEX_DECREMENT 为 1，即缩容 2，对照
            // 容量表，索引量减 2，找到对应的内存，如果实际读取大小小于缩容大小，则进行缩容
            if (actualReadBytes <= SIZE_TABLE[max(0, index - INDEX_DECREMENT - 1)]) {
                // 第一次为 false，然后进入到另一分支，设为 true，即连续两次尝试缩容，第二次
                // 进行缩容
                if (decreaseNow) {
                    // 缩容，容量表对应的索引减一
                    index = max(index - INDEX_DECREMENT, minIndex);
                    // 新的 ByteBuf 大小
                    nextReceiveBufferSize = SIZE_TABLE[index];
                    decreaseNow = false;
                } else {
                    decreaseNow = true;
                }
            } else if (actualReadBytes >= nextReceiveBufferSize) {
                // 扩容，一次加 4，即容量表对应的索引加四
                index = min(index + INDEX_INCREMENT, maxIndex);
                nextReceiveBufferSize = SIZE_TABLE[index];
                decreaseNow = false;
            }
        }

        @Override
        public void readComplete() {
            // 溢出，则使用最大值 Integer.MAX_VALUE，
            record(totalBytesRead());
        }
    }
```
