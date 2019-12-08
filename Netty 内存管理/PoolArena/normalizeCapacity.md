### normalizeCapacity
　　标准化请求的内存容量，为 2 的次方，从大到小来判断。

- 如果是 huge 类型，即大于 16 M，则直接返回，由 JVM 去申请指定容量的的字节数组；
- 大于 512，为 small 和 normal 类型，这两类型的值转换成 2 的幂次方，比如 518，则会找到 1024；
- tiny 类型，如果为 16 的倍数，则直接返回，不为 16，则补齐成 16 倍数的值，返回。

```java
    int normalizeCapacity(int reqCapacity) {
        // 参数校验，不能小于 0
        checkPositiveOrZero(reqCapacity, "reqCapacity");

        // 大于 chunkSize，为 huge 类型，即大于 16M，则直接返回，由 JVM 去申请指定容量的的字节数组
        if (reqCapacity >= chunkSize) {
            return directMemoryCacheAlignment == 0 ? reqCapacity : alignCapacity(reqCapacity);
        }

        // tiny 为 512，大于 tiny 类型即 small 和 normal 类型，这两类型的值转换成
        // 2 的幂次方，比如 518，则会找到 1024
        if (!isTiny(reqCapacity)) {
            // Doubled

            int normalizedCapacity = reqCapacity;
            normalizedCapacity --;
            normalizedCapacity |= normalizedCapacity >>>  1;
            normalizedCapacity |= normalizedCapacity >>>  2;
            normalizedCapacity |= normalizedCapacity >>>  4;
            normalizedCapacity |= normalizedCapacity >>>  8;
            normalizedCapacity |= normalizedCapacity >>> 16;
            normalizedCapacity ++;

            if (normalizedCapacity < 0) {
                normalizedCapacity >>>= 1;
            }
            assert directMemoryCacheAlignment == 0 || (normalizedCapacity & directMemoryCacheAlignmentMask) == 0;

            return normalizedCapacity;
        }

        if (directMemoryCacheAlignment > 0) {
            return alignCapacity(reqCapacity);
        }

        // 如果为 16 的倍数，直接返回
        if ((reqCapacity & 15) == 0) {
            return reqCapacity;
        }

        // 不为 16 倍数的值，则找到比该值大的 16 次方值，比如
        // 38 & ~15 = 32，加上 16 则返回 48，以 16 倍数来自增
        return (reqCapacity & ~15) + 16;
    }
```
