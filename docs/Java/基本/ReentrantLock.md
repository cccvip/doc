上一篇 [Synchorized.md](Synchorized.md) 讲的是 JVM 内置的锁，这篇讲 JUC 里基于 AQS 的 **ReentrantLock**。它和 synchronized 用起来像，但能可中断、能超时、能公平、能有多个条件变量——代价是必须手动 `unlock()`。

这一篇的主线是 **AQS**：搞懂了 AQS 的 `state` 和那条 CLH 变体队列，`ReentrantLock` 就没什么秘密了。

# 总览

ReentrantLock 是 JUC 下基于 AQS 实现的**可重入互斥锁**，实现了 `Lock` 接口。先和 synchronized 对一下：

| | synchronized | ReentrantLock |
|---|---|---|
| 层级 | JVM 关键字 | JDK API |
| 释放 | 自动（异常也释放） | 必须 `finally unlock()` |
| 可中断 | 抢锁时不可中断 | `lockInterruptibly()` |
| 超时 / 尝试 | 不支持 | `tryLock()` / `tryLock(timeout)` |
| 公平锁 | 非公平 | 可选公平 |
| 条件变量 | 一个 wait/notify 集合 | 多个 `Condition` |
| 适用 | 一般同步 | 需要超时、中断、多条件时 |

**优先用 synchronized；需要上面这些高级能力时再用 ReentrantLock。**

# AQS 的两个核心状态

AQS（`AbstractQueuedSynchronizer`）把同步状态和排队都包好了，子类只要实现 `tryAcquire` / `tryRelease` 这类方法：

- `state`：`volatile int`，0 表示未占用，大于 0 表示重入次数；
- `exclusiveOwnerThread`：当前持有锁的线程（来自 `AbstractOwnableSynchronizer`）；
- 等待队列：CLH 变体的**双向链表**，head 是哨兵节点。

![AQS 流程](_assets/img.png)

# lock 流程

加锁的入口是 AQS 的 `acquire`：

```java
public final void acquire(int arg) {
    // 先试一次 tryAcquire；失败才入队，排队期间若被中断则补一次自我中断
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

`tryAcquire` 由子类实现，非公平版的逻辑是：

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 空闲：CAS 抢一下，成功就记下持有线程
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        // 重入：持有者是自己，state + 1
        int nextc = c + acquires;
        if (nextc < 0) throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

抢不到就到这条队列上排队，`acquireQueued` 是排队后的自旋：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {   // 前驱是 head，说明轮到自己了
                setHead(node);                    // 抢到锁，把自己变成新 head
                p.next = null;                    // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed) cancelAcquire(node);
    }
}
```

> 注意第一分支不是"抢锁失败"，而是"排队后轮到自己、恰好能拿到锁"的快速路径。

![acquireQueued](_assets/acquireQueued.png)
![addWaiter](_assets/addWaiter.png)

`shouldParkAfterFailedAcquire` 的作用：第一次进入时把前驱的 `waitStatus` CAS 成 `Node.SIGNAL` 并返回 false；第二次看到 `ws == SIGNAL` 才返回 true，然后 park。也就是说，**"该不该阻塞"要判断两次**——第一次先把责任交给前驱（前驱释放时会通知我），确认之后再安心睡。

![shouldParkAfterFailedAcquire](_assets/shouldParkAfterFailedAcquire.png)

# unlock 流程

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);     // 唤醒后继
        return true;
    }
    return false;
}
```

`tryRelease` 把 `state` 减 1，减到 0 才算真正释放：

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();   // 不是持有者，非法
    boolean free = false;
    if (c == 0) {                                   // 减到 0，彻底释放
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);                                    // 重入时只是把计数减回去
    return free;
}
```

`unparkSuccessor` 负责叫醒队列里第一个没被取消的节点：

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {   // 后继为空或被取消
        s = null;
        // 从尾部往前找第一个有效节点 —— 这也是队列要用双向链表的原因
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

# 公平锁 vs 非公平锁

差别就在"抢之前先不先看队列"：

```java
// 非公平：直接 CAS 抢，不排队
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}

// 公平：tryAcquire 里先问一句"前面还有人排队吗"
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&          // 前面没人排队，才允许抢
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // ... 重入逻辑同非公平
}
```

非公平吞吐更高，但可能让排队线程饥饿；公平锁吞吐低一些，胜在顺序。

# 可重入

同一线程重复 `lock()`，`state` 累加；`unlock()` 递减，减到 0 才真正释放并唤醒后继。这就是上面 `tryAcquire` 里 `current == getExclusiveOwnerThread()` 那条分支。

# Condition

`newCondition()` 返回 `ConditionObject`，内部有自己的条件队列。`await()` 会**完全释放锁**并进入条件队列；`signal()` 把节点从条件队列移到同步队列，等重新抢到锁后 `await` 才返回。

# 实战场景

## 场景一：需要超时或可中断

```java
if (lock.tryLock(3, TimeUnit.SECONDS)) {
    try {
        // 临界区
    } finally {
        lock.unlock();
    }
} else {
    // 获取锁失败，降级或返回
}
```

适合避免死锁、保护下游调用的场景。

## 场景二：多条件变量（有界队列）

```java
class BoundedQueue<T> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity = 10;

    void put(T v) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) notFull.await();
            queue.add(v);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) notEmpty.await();
            T v = queue.poll();
            notFull.signal();
            return v;
        } finally {
            lock.unlock();
        }
    }
}
```

两个 Condition 让生产者和消费者分别等待，避免 `notifyAll` 的无效唤醒。

## 场景三：读写分离

读多写少用 `ReentrantReadWriteLock`：读锁共享、写锁互斥。注意读锁不能升级成写锁（会死锁），写锁可以降级成读锁。

## 场景四：替代 synchronized 的注意点

只有在需要超时、中断、公平、多条件时才换。普通同步用 synchronized 更简单，也不会忘记解锁。

# 常见坑

1. **忘记 unlock**，或 `unlock()` 没放在 `finally`；
2. **`lock()` 放进 try 里**：如果 `lock()` 本身失败，`finally` 里的 `unlock()` 会抛 `IllegalMonitorStateException`。正确做法是 `lock()` 在 try 之前；
3. **`await()` 用 `if` 而不是 `while`**：虚假唤醒后条件已不满足；
4. **`await()` / `signal()` 不在锁内**：抛 `IllegalMonitorStateException`；
5. **默认用公平锁**：吞吐明显下降，除非确实需要顺序；
6. **以为能跨 JVM**：ReentrantLock 只在单 JVM 内有效，分布式锁要用 Redis / ZooKeeper / DB。

# 小结

1. **ReentrantLock 和 AQS 的关系？** ReentrantLock 把同步逻辑委托给内部类 `Sync`（继承 AQS），公平/非公平分别是 `FairSync` / `NonfairSync`，重写 `tryAcquire` / `tryRelease`。
2. **state 的作用？** 表示锁的占用状态和重入次数：0 空闲，大于 0 被占用且数值是重入次数；通过 CAS 修改保证并发安全。
3. **公平锁和非公平锁的区别？** 非公平锁允许插队（`lock()` 先直接 CAS 抢一次），吞吐高但可能饥饿；公平锁用 `hasQueuedPredecessors()` 检查队列，保证先来先得。
4. **可重入怎么实现的？** `state` 计数：同一线程再次加锁 `state + 1`，解锁 `- 1`，减到 0 才释放锁。
5. **lock、tryLock、lockInterruptibly 的区别？** `lock()` 阻塞直到拿到锁、不可中断；`tryLock()` 立即返回成功/失败；`tryLock(timeout)` 最多等一段时间；`lockInterruptibly()` 等待时可被中断。
6. **Condition 的原理？** `await()` 完全释放锁并进入条件队列；`signal()` 把节点移到同步队列，等重新抢到锁后 `await` 才返回。
7. **为什么 unlock 必须放 finally？** 锁不会自动释放；临界区抛异常又没写 finally，就会永久占锁。
8. **为什么非公平锁性能更好？** 新来的线程直接 CAS 抢，抢到就执行，省去唤醒排队线程的上下文切换；代价是可能插队。
9. **AQS 的队列为什么用双向链表？** `unparkSuccessor` 在 `next` 不可靠时会从 `tail` 反向扫描找有效节点，所以要有 `prev`；入队、取消、唤醒也都需要前后指针。

# 参考

- JDK8 `ReentrantLock`、`AbstractQueuedSynchronizer` 源码
- [Synchorized.md](Synchorized.md)（synchronized 与锁升级）
