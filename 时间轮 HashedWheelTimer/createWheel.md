### HashedWheelBucket

```java
    private static HashedWheelBucket[] createWheel(int ticksPerWheel) {
        if (ticksPerWheel <= 0) {
            throw new IllegalArgumentException(
                    "ticksPerWheel must be greater than 0: " + ticksPerWheel);
        }
        if (ticksPerWheel > 1073741824) {
            throw new IllegalArgumentException(
                    "ticksPerWheel may not be greater than 2^30: " + ticksPerWheel);
        }
        // 对时间轮个数进行标准化，需为 2 的次方
        ticksPerWheel = normalizeTicksPerWheel(ticksPerWheel);
        // 创建 HashedWheelBucket 的数组
        HashedWheelBucket[] wheel = new HashedWheelBucket[ticksPerWheel];
        for (int i = 0; i < wheel.length; i ++) {
            wheel[i] = new HashedWheelBucket();
        }
        return wheel;
    }
```

#### normalizeTicksPerWheel
　　不断左移一位，直到找到比 ticksPerWheel 大的 2 次方值，比如 3，则最近的二次方值为 4。注意，如果 ticksPerWheel 值很大，这个方法会循环很多次。

```java
    private static int normalizeTicksPerWheel(int ticksPerWheel) {
        int normalizedTicksPerWheel = 1;
        while (normalizedTicksPerWheel < ticksPerWheel) {
            normalizedTicksPerWheel <<= 1;
        }
        return normalizedTicksPerWheel;
    }
```

#### normalizeTicksPerWheel
　　可参考 JDK 的 hashmap 算法，同样也是寻找比 ticksPerWheel 大的 2 次方值。
```java
private int normalizeTicksPerWheel(int ticksPerWheel) {
    int n = ticksPerWheel - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    // 1073741824 = 2^30，防止溢出
    return (n < 0) ? 1 : (n >= 1073741824) ? 1073741824 : n + 1;
}
```