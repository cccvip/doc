`synchronized` 是每个人最先学的锁，但真到面试里追问"锁的到底是什么""偏向锁呢""凭什么保证可见性"，很多人就答不完整了。这篇从对象头一路讲到字节码，把这几件事一次说清。

> 关于 volatile、内存屏障、happens-before 这些"可见性"的底层，本文不展开，集中放在 [可见性.md](可见性.md) 里讲。

# 总览

一句话记住：**synchronized 锁的是对象（或 Class），不是代码块本身。**

它保证三件事：**原子性、可见性、有序性**。

# 锁的是什么？

| 用法 | 锁对象 |
|---|---|
| `synchronized` 实例方法 | 当前实例 `this` |
| `synchronized` 静态方法 | 当前类的 `Class` 对象 |
| `synchronized(obj)` 代码块 | 括号里的 `obj` |

所以"锁对象不一致 = 等于没锁"：两个方法一个锁 `this`、一个锁 `new Object()`，其实是两把锁。

# 从字节码看加锁

同步代码块编译后是 `monitorenter` / `monitorexit`：

```java
public void sync() {
    synchronized (this) {
        int a = 1;
    }
}
```

反编译出来大致是这样（`javap -c`）：

```text
public void sync();
  Code:
     0: aload_0
     1: dup
     2: astore_1
     3: monitorenter        // 进入同步块：加锁
     4: iconst_1
     5: istore_2
     6: aload_1
     7: monitorexit         // 正常退出：解锁
     8: goto 16
    11: aload_1
    12: monitorexit         // 异常退出：也要解锁
    13: athrow
```

关键点：**为什么有两个 `monitorexit`**？因为编译器给正常路径和异常路径各生成一个，保证**即使临界区抛异常，锁也一定被释放**——这正是 synchronized 比手动锁省心的地方（ReentrantLock 得自己写 `finally`）。

同步方法则不一样，它是靠方法访问标志 `ACC_SYNCHRONIZED`，JVM 调用方法时隐式加锁、退出时解锁，没有显式的 `monitorenter`。

锁的载体是**对象头的 Mark Word**；膨胀成重量级锁后，Mark Word 指向 `ObjectMonitor`。

# 对象头与锁升级

看 Mark Word 的低位（锁标志位）就能判断当前是哪种锁：

| 状态 | 标志位 | 说明 |
|---|---|---|
| 无锁 | `001` | |
| 偏向锁 | `101` | Mark Word 记录持有线程 ID |
| 轻量级锁 | `00` | Mark Word 指向栈帧里的 Lock Record |
| 重量级锁 | `10` | Mark Word 指向 `ObjectMonitor` |
| GC 标记 | `11` | 兼作 GC 标记用 |

升级路线：**无锁 → 偏向锁 → 轻量级锁 → 重量级锁**，且基本不可逆。

- **偏向锁**：无实际竞争、且将来只有第一个申请锁的线程会用它。Mark Word 记录持有线程 ID，同一线程再次进入无需 CAS。注意：**偏向锁本身不维护重入计数**，可重入是升级到轻量级锁之后靠 Lock Record 计数的。JDK15 起已默认禁用（JEP 374）。
- **轻量级锁**：无实际竞争、多个线程交替用锁、允许短时间竞争。线程在自己栈帧里建 Lock Record，用 CAS 把 Mark Word 换成指向 Lock Record 的指针；CAS 失败就自旋重试。
- **重量级锁**：有实际竞争且竞争时间长。依赖 `ObjectMonitor`，抢不到锁的线程会被 park 阻塞，涉及用户态/内核态切换，开销最大。

什么时候升级：线程 2 来抢锁，发现已被线程 1 持有 → 偏向锁升级为轻量级锁；自旋多次仍抢不到 → 升级为重量级锁。可以类比成：**一个人用卫生间；第二个人不停敲门问好了没；一直问不到，干脆把门锁死、让后来的人排队。**

# 三个特性怎么保证？

- **原子性**：临界区代码互斥执行，同一时刻只有一个线程能进来；
- **可见性**：`monitorexit` 前的写会刷回主内存；happens-before 规定「对一个锁的 unlock 先于后续的 lock」（底层细节见 [可见性.md](可见性.md)）；
- **有序性**：临界区内外的指令不会越过 monitor 边界重排。

# wait / notify

- 必须在 `synchronized` 块内调用，否则抛 `IllegalMonitorStateException`（要先持有 monitor）；
- `wait()` 会**释放锁**并进入等待；`notify()` 只唤醒一个等待线程，**不释放锁**，等当前同步块执行完才释放；
- 用来做线程间协作，典型是生产者-消费者。

# 实战场景

## 场景一：DCL 单例

```java
public class Singleton {
    private static volatile Singleton singleton;

    private Singleton() {}

    public static Singleton getInstance() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

面试陷阱：不加 volatile 会有问题吗？

JVM 中 `new` 一个对象大致分四步：①分配内存；②内存清零；③设置对象头；④执行构造方法 `<init>`。指令重排后，可能先把引用赋给 `singleton`，再执行构造方法：

> 线程 A 停在第 ③ 步，引用已非空、但构造方法还没跑完；线程 B 判断 `singleton == null` 得到 false，直接拿去用，此时属性还是默认值。

volatile 的作用就是禁止「引用赋值」和「构造方法执行」重排。

## 场景二：线程安全计数器 / 库存扣减

```java
public class Counter {
    private int count;
    public synchronized void incr() { count++; }
    public synchronized int get() { return count; }
}
```

```java
public synchronized boolean deduct(int n) {
    if (stock < n) return false;
    stock -= n;
    return true;
}
```

注意：synchronized 只保证**单个 JVM 内**的互斥。多实例部署时要靠 Redis 原子操作或数据库行锁。

## 场景三：生产者-消费者（wait/notify）

```java
class Buffer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 10;

    public synchronized void put(int v) throws InterruptedException {
        while (queue.size() == capacity) wait();   // 用 while，不用 if
        queue.add(v);
        notifyAll();
    }

    public synchronized int take() throws InterruptedException {
        while (queue.isEmpty()) wait();
        int v = queue.poll();
        notifyAll();
        return v;
    }
}
```

两个细节：条件判断用 `while` 而不是 `if`（防虚假唤醒）；一般用 `notifyAll` 而不是 `notify`（避免只唤醒同类线程互相等待）。

## 场景四：锁对象的选择

```java
private static final Object LOCK = new Object();
```

推荐用私有的专用锁对象。不要锁 `String` 字面量、`Integer` 缓存对象、外部可访问的 `this` 或共享对象——它们可能被别处当成同一把锁，造成意外互斥甚至死锁。

## 场景五：多把锁与死锁

转账场景：A 锁自己再锁 B，另一个线程 B 锁自己再锁 A，就会循环等待。解决：**按账户 id 排序后依次加锁**，让所有线程加锁顺序一致。

## 场景六：降低锁粒度

一把大锁会让所有请求串行。可以把锁拆细：ConcurrentHashMap（JDK7 的 Segment 分段锁）、按 key 哈希分桶加锁、读写分离用 `ReentrantReadWriteLock`。核心思想是**缩小临界区、减少共享**。

# 常见坑

1. **锁对象不一致**：锁 `this` 和方法里 `new Object()` 是两把锁，等于没锁；
2. **锁 String 常量 / 包装类**：常量池和缓存会复用对象，可能和无关代码互斥；
3. **锁粒度过大**：把 IO、RPC、睡眠放进同步块，吞吐骤降；
4. **`if` 代替 `while`**：虚假唤醒后条件已不满足，仍继续执行；
5. **多锁顺序不一致**：死锁；
6. **误以为能跨实例 / 跨 JVM**：synchronized 只对同一个 JVM 里的同一个对象生效；
7. **同步块里调用外部方法**：可能触发回调、长时间阻塞，甚至反向加锁。

# 小结

1. **synchronized 的原理？** JVM 通过对象头 Mark Word + monitor 实现：代码块用 `monitorenter/monitorexit`，方法用 `ACC_SYNCHRONIZED` 标志；重量级锁最终落到 `ObjectMonitor` 的 enter/exit。
2. **锁升级的过程？** 无锁 → 偏向锁（单线程重入）→ 轻量级锁（交替执行、短竞争、CAS 自旋）→ 重量级锁（激烈竞争、阻塞排队）；基本不可逆。
3. **synchronized 和 ReentrantLock 的区别？** 前者是 JVM 关键字、自动释放、非公平、抢锁不可中断、只有一个等待集；后者是 JDK API、必须 `finally unlock`、可选公平、可中断、可超时、支持多个 `Condition`。
4. **是可重入的吗？** 是。重量级锁靠 `ObjectMonitor` 的重入计数，轻量级锁靠栈帧里的 Lock Record，偏向锁直接判断线程 ID。
5. **能保证可见性吗？** 能。释放锁前的写会刷回主内存，后续获取同一把锁的线程能看到；本质是 unlock 与 lock 之间的 happens-before。
6. **能保证原子性吗？** 只保证**临界区内**的原子性。多个方法组合、或锁的不是同一个对象时，仍不安全。
7. **是公平锁吗？** 不是。非公平，允许插队，吞吐更高，但可能造成饥饿。
8. **可以被中断吗？** 阻塞在抢锁上时**不响应中断**；`ReentrantLock.lockInterruptibly()` 可以。
9. **锁消除、锁粗化、自适应自旋？** 锁消除：JIT 经逃逸分析发现锁对象不逃逸就去掉锁；锁粗化：把相邻多次加锁合并成一次；自适应自旋：按上次自旋成功率动态决定自旋次数。
10. **wait/notify 为什么必须在 synchronized 里？** 它们要求先持有对象的 monitor，否则抛 `IllegalMonitorStateException`；同时可避免「检查条件」和「进入等待」之间的 lost wakeup 竞态。
11. **死锁的四个条件？** 互斥、请求并保持、不可剥夺、循环等待。破坏任意一个即可，工程上最常用**固定加锁顺序**。

# 参考

- JDK8 `ObjectMonitor`、`synchronized` 字节码
- JEP 374: Deprecate and Disable Biased Locking
- [可见性.md](可见性.md)（内存语义、内存屏障）
