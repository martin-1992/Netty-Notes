### InternalThreadLocalMap#get

- 当前线程为 FastThreadLocalThread（Netty 封装的线程），使用 fastGet() 方法直接获取 threadLocalMap；
- 当前线程为普通线程，使用 slowGet() 获取 slowThreadLocalMap 的 InternalThreadLocalMap。

　　Netty 的 FastThreadLocalThread 高性能之一是其底层使用数组形式，且大小为 2 的次方，而普通线程的 InternalThreadLocalMap 底层使用 HashMap。

```java
    public static InternalThreadLocalMap get() {
        Thread thread = Thread.currentThread();
        if (thread instanceof FastThreadLocalThread) {
            return fastGet((FastThreadLocalThread) thread);
        } else {
            return slowGet();
        }
    }
```

### InternalThreadLocalMap#fastGet
　　为 Netty 封装的线程，获取 FastThreadLocalThread 的 InternalThreadLocalMap。如果为空，则创建一个容量为 32 的对象数组，并使用变量 threadLocalMap 保存该对象数组。

```java
    private static InternalThreadLocalMap fastGet(FastThreadLocalThread thread) {
        InternalThreadLocalMap threadLocalMap = thread.threadLocalMap();
        if (threadLocalMap == null) {
            // 创建 InternalThreadLocalMap
            thread.setThreadLocalMap(threadLocalMap = new InternalThreadLocalMap());
        }
        return threadLocalMap;
    }
```

#### InternalThreadLocalMap 的构造函数
　　创建一个容量大小为 32 的对象数组，填充默认值 Object 对象 UNSET，并使用变量 indexedVariables 保存创建的对象数组。

```java
    // 默认值 UNSET
    public static final Object UNSET = new Object();

    private InternalThreadLocalMap() {
        // 使用变量 indexedVariables 保存创建的对象数组
        super(newIndexedVariableTable());
    }
        
    private static Object[] newIndexedVariableTable() {
        // 对象数组大小为 32
        Object[] array = new Object[32];
        // 填充默认值 UNSET
        Arrays.fill(array, UNSET);
        return array;
    }
    
    UnpaddedInternalThreadLocalMap(Object[] indexedVariables) {
        this.indexedVariables = indexedVariables;
    }
```

### InternalThreadLocalMap#slowGet

- slowGet()，获取的 InternalThreadLocalMap 是 HashMap 结构的；
- 从 HashMap 根据当前线程 thread 获取对应的 InternalThreadLocalMap；
- 获取不到，则创建一个容量为 32 的对象数组，并使用变量 indexedVariables 保存该对象数组，这与 fastGet 一样，创建的是数组形式的 InternalThreadLocalMap。

```java
    static final ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = new ThreadLocal<InternalThreadLocalMap>();

    private static InternalThreadLocalMap slowGet() {
        ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = UnpaddedInternalThreadLocalMap.slowThreadLocalMap;
        // 从 HashMap 根据当前线程 thread 获取对应的 InternalThreadLocalMap
        InternalThreadLocalMap ret = slowThreadLocalMap.get();
        if (ret == null) {
            // 创建 InternalThreadLocalMap，用的是 fastGet 的创建方法，即创建数组形式的 InternalThreadLocalMap
            ret = new InternalThreadLocalMap();
            slowThreadLocalMap.set(ret);
        }
        return ret;
    }
```
