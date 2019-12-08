### get

- 调用 [get](https://github.com/martin-1992/Netty-Notes/blob/master/FastThreadLocal/InternalThreadLocalMap/get.md) 获取当前线程实例对应的 InternalThreadLocalMap;
- indexedVariable()，为当前线程对应的 InternalThreadLocalMap 中的一个 Object 数组，存储本地变量。一个 FastThreadLoacal 实例绑定一个 index 索引值，根据索引值获取数组中对应的变量值；
- 如果获取的值为默认值 UNSET，则调用 initialize 方法初始化 InternalThreadLocalMap，将值插入到数组 InternalThreadLocalMap 中。

```java
    @SuppressWarnings("unchecked")
    public final V get() {
        // 获取当前线程实例对应的 InternalThreadLocalMap
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
        Object v = threadLocalMap.indexedVariable(index);
        if (v != InternalThreadLocalMap.UNSET) {
            return (V) v;
        }
        // 为 UNSET，表示索引值大于数组长度，可能是未初始化 threadLocalMap，先对其初始化
        return initialize(threadLocalMap);
    }
        
    @SuppressWarnings("unchecked")
    public final V get(InternalThreadLocalMap threadLocalMap) {
        Object v = threadLocalMap.indexedVariable(index);
        if (v != InternalThreadLocalMap.UNSET) {
            return (V) v;
        }

        return initialize(threadLocalMap);
    }
```

#### FastThreadLocal#initialize
　　初始化，调用 [setIndexedVariable](https://github.com/martin-1992/Netty-Notes/blob/master/FastThreadLocal/InternalThreadLocalMap/setIndexedVariable.md) 将初始值插入数组 indexedVariables 中。

```java
    private V initialize(InternalThreadLocalMap threadLocalMap) {
        V v = null;
        try {
            // 由用户重写该方法，自定义初始值
            v = initialValue();
        } catch (Exception e) {
            PlatformDependent.throwException(e);
        }
        // 将值 v 插入对象数组中
        threadLocalMap.setIndexedVariable(index, v);
        addToVariablesToRemove(threadLocalMap, this);
        return v;
    }
    
    /**
     * 重写该方法，获取初始值
     **/
    protected V initialValue() throws Exception {
        return null;
    }
```
