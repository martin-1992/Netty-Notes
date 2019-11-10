### DefaultChannelPipeline#checkMultiplicity
　　判断 ChannelHandler 是否能重复添加，即一个 ChannelHandler 添加到多个 Channel 里。如果 Handler 不为共享式，且已添加，则抛出异常。

```java
    private static void checkMultiplicity(ChannelHandler handler) {
        if (handler instanceof ChannelHandlerAdapter) {
            ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
            // 判断 Handler 是否为共享式的，如不为共享，且已添加，则抛出异常
            if (!h.isSharable() && h.added) {
                throw new ChannelPipelineException(
                        h.getClass().getName() +
                        " is not a @Sharable handler, so can't be added or removed multiple times.");
            }
            // handler 已被添加
            h.added = true;
        }
    }
```


### ChannelHandlerAdapter#isSharable
　　判断此 ChannelHandlerAdapter 是否为共享式的，如果能拿到 Sharable 注解，则为共享式，可以添加到多个 Channel，否则返回 false。

```java
    public boolean isSharable() {

        Class<?> clazz = getClass();
        // 从缓存中获取
        Map<Class<?>, Boolean> cache = InternalThreadLocalMap.get().handlerSharableCache();
        Boolean sharable = cache.get(clazz);
        if (sharable == null) {
            // 获取该类的 Sharable 注解，如果有注解的 Sharable，说明这个 Handler 可被多个 Channel 共享，
            // 即可多次添加。如果不是，则返回 false
            sharable = clazz.isAnnotationPresent(Sharable.class);
            cache.put(clazz, sharable);
        }
        return sharable;
    }
```

