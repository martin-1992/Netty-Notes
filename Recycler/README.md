### Recycler
　　Recycler 为 Netty 的轻量级对象池，用于获取对象和用完回收对象，其优势：

- **可复用对象，** 省去了频繁创建、销毁对象的开销；
- **减少 young GC，** 因为减少了不必要的对象创建，因此 JVM 年轻代垃圾减少。

### [get](https://github.com/martin-1992/Netty-Notes/blob/master/Recycler/get.md)
　　从对象池中获取对象，获取不到对象时会调用 [scavenge](https://github.com/martin-1992/Netty-Notes/blob/master/Recycler/scavenge.md) 方法从当前线程 Stack 对应的 WeakOrderQueue 链表回收 DefaultHandle 对象。

### [recycle](https://github.com/martin-1992/Netty-Notes/blob/master/Recycler/recycle.md)
　　回收对象有两种：

- 同线程回收对象，即对象在一个线程中分配，同时在该线程中回收对象；
- 异线程回收对象，即对象在一个线程中分配，但在另一个线程中回收对象。
