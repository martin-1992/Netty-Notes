### FastThreadLocal 分析
　　Netty 中 FastThreadLocal 用来代替 ThreadLocal 存放线程本地变量，FastThreadLocal 相比 ThreadLocal 有两个优势。

- 原先的 ThreadLoacal 使用弱引用的 ThreadLoacalMap，存在内存泄漏的可能。Netty 不使用弱引用的 ThreadLocalMap，不存在该问题；
- FastThreadLocal 使用数组来存放变量，**原生的 ThreadLocal 底层也是数组形式，但是使用哈希加线性探测法来实现的，** 当数据量大时或遇到哈希冲突时，ThreadLocal 没有 FastThreadLocal 快。

　　FastThreadLocal 使用 InternalThreadLocalMap 来存放数据，和 ThreadLocal 实现方式类似，FastThreadLocalThread 中有一个 InternalThreadLocalMap 类型的字段 threadLocalMap，这样一个线程对应一个 InternalThreadLocalMap 实例，该线程下所有的线程本地变量都会放在 threadLocalMap 中的数组 indexedVariables 中。

### FastThreadLocal 的构造函数

- 每个 FastThreadLocal 实例对应一个唯一的 index，当一个 FastThreadLocalThread 线程通过 FastThreadLocal 获取线程本地变量时，是通过 FastThreadLocal 对应的 index 变量值，在 FastThreadLocalThread 的 threadLocalMap 中的数组 indexedVariables 中以该 index 为索引得到的；
- InternalThreadLocalMap.nextVariableIndex() 为静态方法，维护了一个原子类 nextIndex，保证线程安全。

```java
    // 每个 FastThreadLocal 实例对应一个唯一的 index
    private final int index;

    public FastThreadLocal() {
        index = InternalThreadLocalMap.nextVariableIndex();
    }
```


#### InternalThreadLocalMap#nextVariableIndex
　　静态方法，维护了一个原子类 nextIndex，

```java
    static final AtomicInteger nextIndex = new AtomicInteger();

    public static int nextVariableIndex() {
        int index = nextIndex.getAndIncrement();
        if (index < 0) {
            nextIndex.decrementAndGet();
            throw new IllegalStateException("too many thread-local indexed variables");
        }
        return index;
    }
```

### [get](https://github.com/martin-1992/Netty-Notes/blob/master/FastThreadLocal/get.md)
　　获取当前线程的 InternalThreadLocalMap 存储的对象。

### [set](https://github.com/martin-1992/Netty-Notes/blob/master/FastThreadLocal/set.md)
　　将对象存储到当前线程的 InternalThreadLocalMap 中。
