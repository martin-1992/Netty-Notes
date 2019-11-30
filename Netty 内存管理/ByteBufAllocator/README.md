### ByteBufAllocator
　　负责分配所有 ByteBuf 类型的内存，为 Netty 的内存管理器。ByteBufAllocator 为接口，默认实现为 [AbstractByteBufAllocator]()。

- AbstractByteBufAllocator，实现 ByteBufAllocator 的大部分功能，暴露出两个抽象方法 newHeapBuffer 和 newDirectBuffer，让子类实现，为创建堆内和堆外逻辑；
- PooledByteBufAllocator，是在预先分配好的内存里取一段。实现 newHeapBuffer 和 newDirectBuffer，又分堆外和堆内；
- UnpooledByteBufAllocator，直接调用系统 API 申请分配一块内存。实现 newHeapBuffer 和 newDirectBuffer，又分堆外和堆内。

```java
public interface ByteBufAllocator {

    ByteBufAllocator DEFAULT = ByteBufUtil.DEFAULT_ALLOCATOR;

    /**
     * 分配内存，堆内或堆外内存，由子类实现
     */
    ByteBuf buffer();

    ByteBuf buffer(int initialCapacity);

    ByteBuf buffer(int initialCapacity, int maxCapacity);

    /**
     * 在分配 ioBuffer 时，如果能分配 direct，就分配，因为它更适合于 IO。
     */
    ByteBuf ioBuffer();

    ByteBuf ioBuffer(int initialCapacity);

    ByteBuf ioBuffer(int initialCapacity, int maxCapacity);

    /**
     * 在堆上进行内存分配
     */
    ByteBuf heapBuffer();

    ByteBuf heapBuffer(int initialCapacity);

    ByteBuf heapBuffer(int initialCapacity, int maxCapacity);

    /**
     * 在堆外进行内存分配
     */
    ByteBuf directBuffer();

    ByteBuf directBuffer(int initialCapacity);

    ByteBuf directBuffer(int initialCapacity, int maxCapacity);

    /**
     * 在分配 ByteBuf 时，其底层可为 direct 和 heap 的组合
     */
    CompositeByteBuf compositeBuffer();

    CompositeByteBuf compositeBuffer(int maxNumComponents);

    CompositeByteBuf compositeHeapBuffer();

    CompositeByteBuf compositeHeapBuffer(int maxNumComponents);

    CompositeByteBuf compositeDirectBuffer();

    CompositeByteBuf compositeDirectBuffer(int maxNumComponents);

    boolean isDirectBufferPooled();

    int calculateNewCapacity(int minNewCapacity, int maxCapacity);
 }
 ```


