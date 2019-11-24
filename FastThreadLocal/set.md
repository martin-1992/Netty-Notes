### set

- 判断该值不为默认值 UNSET，则获取当前线程的 InternalThreadLocalMap；
- 将值插入到 InternalThreadLocalMap 中。

```java
    public static final Object UNSET = new Object();

    public final void set(V value) {
        if (value != InternalThreadLocalMap.UNSET) {
            InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
            setKnownNotUnset(threadLocalMap, value);
        } else {
            remove();
        }
    }
    
    public final void set(InternalThreadLocalMap threadLocalMap, V value) {
        if (value != InternalThreadLocalMap.UNSET) {
            setKnownNotUnset(threadLocalMap, value);
        } else {
            remove(threadLocalMap);
        }
    }
    
    private void setKnownNotUnset(InternalThreadLocalMap threadLocalMap, V value) {
        if (threadLocalMap.setIndexedVariable(index, value)) {
            addToVariablesToRemove(threadLocalMap, this);
        }
    }
```



#### addToVariablesToRemove
　　将 FastThreadLocal 添加到 Set 中，因为 Netty 的 Map 只是一个数组，没有键，所以保存到一个 Set 中，这样就可以判断是否 set 过这个 map，例如 Netty 的 isSet 方法就是根据这个判断的。

```java
    static final AtomicInteger nextIndex = new AtomicInteger();
    private static final int variablesToRemoveIndex = InternalThreadLocalMap.nextVariableIndex();

    @SuppressWarnings("unchecked")
    private static void addToVariablesToRemove(InternalThreadLocalMap threadLocalMap, FastThreadLocal<?> variable) {
        // 获取该索引 variablesToRemoveIndex 对应的值
        Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);
        Set<FastThreadLocal<?>> variablesToRemove;
        // 如果该值为空，创建一个基于 IdentityHashMap 的 Set
        if (v == InternalThreadLocalMap.UNSET || v == null) {
            variablesToRemove = Collections.newSetFromMap(new IdentityHashMap<FastThreadLocal<?>, Boolean>());
            // 将创建的 set（variablesToRemove）保存到数组中该索引 variablesToRemoveIndex 的位置
            threadLocalMap.setIndexedVariable(variablesToRemoveIndex, variablesToRemove);
        } else {
            // 不为空，则转为 set
            variablesToRemove = (Set<FastThreadLocal<?>>) v;
        }
        // 将变量值 FastThreadLocal 添加到 set 中
        variablesToRemove.add(variable);
    }
```

#### InternalThreadLocalMap#nextVariableIndex
　　获取下个值。

```java
    public static int nextVariableIndex() {
        int index = nextIndex.getAndIncrement();
        if (index < 0) {
            nextIndex.decrementAndGet();
            throw new IllegalStateException("too many thread-local indexed variables");
        }
        return index;
    }
```