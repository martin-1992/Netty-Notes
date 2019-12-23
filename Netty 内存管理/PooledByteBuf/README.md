### PooledByteBuf
　　Pooled 为池化，即使用对象池，可重用的 ByteBuf。当对象用完释放后会放回到对象池中，不需要频繁创建对象。

#### 构造函数
　　设置 recyclerHandle，Handle 只有一个方法，即 recycle，用于回收对象。

```java
abstract class PooledByteBuf<T> extends AbstractReferenceCountedByteBuf {

    // 用于回收 ByteBuf
    private final Recycler.Handle<PooledByteBuf<T>> recyclerHandle;
    // 用于分配内存块
    protected PoolChunk<T> chunk;
    // 分配内存块所处的位置
    protected long handle;

    protected T memory;
    // 偏移量
    protected int offset;
    // 容量
    protected int length;
    // memory 的大小
    int maxLength;
    // 内存分配器
    PoolThreadCache cache;
    // 临时 ByteBuff 对象
    ByteBuffer tmpNioBuf;
    // ByteBuf 分配器对象
    private ByteBufAllocator allocator;

    @SuppressWarnings("unchecked")
    protected PooledByteBuf(Recycler.Handle<? extends PooledByteBuf<T>> recyclerHandle, int maxCapacity) {
        super(maxCapacity);
        this.recyclerHandle = (Handle<PooledByteBuf<T>>) recyclerHandle;
    }
```

### init0
　　 初始化 PooledByteBuf 对象，使用变量保存传进来的参数，包括缓存的内存块地址 handle、chunk、偏移量 offset 等。

```java
    void init(PoolChunk<T> chunk, ByteBuffer nioBuffer,
              long handle, int offset, int length, int maxLength, PoolThreadCache cache) {
        init0(chunk, nioBuffer, handle, offset, length, maxLength, cache);
    }

    void initUnpooled(PoolChunk<T> chunk, int length) {
        init0(chunk, null, 0, chunk.offset, length, length, null);
    }

    private void init0(PoolChunk<T> chunk, ByteBuffer nioBuffer,
                       long handle, int offset, int length, int maxLength, PoolThreadCache cache) {
        assert handle >= 0;
        // 要通过 chunk 来分配内存
        assert chunk != null;
        // 如果是通过缓存来分配的，通过 chunk 和 handle，则能确定一块内存
        this.chunk = chunk;
        // 在分配 chunk 时，通过调用 JDK API 申请的一块 DirectByteBuf
        memory = chunk.memory;
        tmpNioBuf = nioBuffer;
        allocator = chunk.arena.parent;
        this.cache = cache;
        // chunk 分配的内存块所处的位置，即缓存的内存块地址
        this.handle = handle; 
        // page 级别的偏移量 offset 为 0
        this.offset = offset;
        // 保存请求容量 reqCapacity
        this.length = length;
        // 保存 subpage.elemSize
        this.maxLength = maxLength;
    }
```

### reuse
　　重复使用创建的对象，需要先将已创建对象进行重置。

```java
    /**
     * 设置前面分析 ByteBuf 的一些常用变量
     * Method must be called before reuse this {@link PooledByteBufAllocator}
     */
    final void reuse(int maxCapacity) {
        // 最大扩容容量
        maxCapacity(maxCapacity);
        // 设置应用，表示当前的 ByteBuf 被多个地方进行引用
        setRefCnt(1);
        // 读写指针
        setIndex0(0, 0);
        // 回收站里跟标记相关的值，进行重置
        discardMarks();
    }
```
