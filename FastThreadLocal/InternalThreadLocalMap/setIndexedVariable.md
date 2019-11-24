### InternalThreadLocalMap#setIndexedVariable
　　设置新的线程局部变量，如果索引在数组范围内，则覆盖旧值，否则进行扩容。

```java
    public boolean setIndexedVariable(int index, Object value) {
        // 在 InternalThreadLocalMap#get 方法中，如果没有获取到 InternalThreadLocalMap，会创建一个默认大小为
        // 32 的对象数组，使用 indexedVariables 变量保存，将其包装成 InternalThreadLocalMap
        Object[] lookup = indexedVariables;
        // 设置新值
        if (index < lookup.length) {
            Object oldValue = lookup[index];
            lookup[index] = value;
            // 如果要覆盖的值不为默认值 UNSET，则返回 false
            return oldValue == UNSET;
        } else {
            // 两倍扩容，将旧数组的值复制到新数组
            expandIndexedVariableTableAndSet(index, value);
            return true;
        }
    }
```


### InternalThreadLocalMap#expandIndexedVariableTableAndSet
　　两倍扩容，将旧数组的值复制到新数组中。这里，使用位运算，进行两倍扩容，与 HashMap 的扩容方法类似。

```java
    private void expandIndexedVariableTableAndSet(int index, Object value) {
        Object[] oldArray = indexedVariables;
        final int oldCapacity = oldArray.length;
        int newCapacity = index;
        newCapacity |= newCapacity >>>  1;
        newCapacity |= newCapacity >>>  2;
        newCapacity |= newCapacity >>>  4;
        newCapacity |= newCapacity >>>  8;
        newCapacity |= newCapacity >>> 16;
        newCapacity ++;

        Object[] newArray = Arrays.copyOf(oldArray, newCapacity);
        // 旧数组的值复制到新数组的值，剩余的填充 UNSET
        Arrays.fill(newArray, oldCapacity, newArray.length, UNSET);
        // 设置新值
        newArray[index] = value;
        // 保存新数组 InternalThreadLocalMap
        indexedVariables = newArray;
    }
```

