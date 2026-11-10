# what

ThreadLocal是Java中的一个类，用于创建线程局部变量。它允许每个线程拥有自己独立的变量副本，而不会相互干扰。通常情况下，如果多个线程共享同一个变量，会导致线程安全问题。但是通过ThreadLocal，每个线程可以访问自己的变量副本，从而避免了线程安全问题。

使用ThreadLocal，可以在每个线程中存储和获取特定的值，而不需要使用同步机制。每个线程都可以独立地修改自己的变量副本，而不会影响其他线程的副本。

ThreadLocal 常被用来为每个线程维护一份独立实例——注意这**不是"单例"**，而是"每线程一份"。把它当单例用要小心：它不保证全局唯一，线程越多实例越多，用完还要记得 `remove()`。

需要注意的是，使用ThreadLocal时要注意内存泄漏问题。由于ThreadLocal是线程级别的变量，如果在使用完毕后没有手动清除，可能会导致对象无法被垃圾回收，进而造成内存泄漏。因此，使用ThreadLocal时应该及时清理不再使用的变量副本。


# how

![img_1.png](threadLocal.png)

- 斐波拉契散列

使用如下的几张方式计算hash散列

```shell
int HASH_INCREMENT=0x61c88647
hashCode = (i * HASH_INCREMENT + HASH_INCREMENT)&(size-1);
```

- 存储结构
ThreadLocal 本身不存数据，数据存在每个线程自己的 ThreadLocalMap 里（即 `Thread.threadLocals`）。这个 Map 的 key 是 **ThreadLocal 实例本身**（弱引用），value 才是你要存的泛型值。

- 内存泄漏

内存泄漏：指程序中已经动态分配的堆内存由于某种原因无法释放,造成系统内存的浪费，导致程序运行速度减缓甚至系统崩溃。

为什么会造成内存泄漏？

ThreadLocalMap 的 Entry 继承自 WeakReference：key（ThreadLocal 实例）是弱引用，value 是强引用。

泄漏的典型场景**不是线程结束**，而是**线程长期存活**（比如线程池里的工作线程）：

1. ThreadLocal 实例被外部回收后，Entry 的 key 变成 null；
2. 但这条强引用链还在：`Thread → ThreadLocalMap → Entry → value`；
3. 只要线程不死，value 就无法被 GC，直到后续 get/set/remove 触发 `expungeStaleEntry` 或扩容时才被顺带清理。

如果线程真的结束了，`threadLocals` 整体不可达，反而会被正常回收。所以规范做法是用完调用 `remove()`，尤其是在线程池场景。







