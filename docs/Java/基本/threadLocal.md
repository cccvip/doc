ThreadLocal 用起来简单，但一追问"数据到底存在哪""为什么 key 用弱引用""内存泄漏是怎么漏的"，很多人就说不清了。这篇不背结论，直接从 `ThreadLocalMap` 的源码看进去。

> 本文对着 JDK8 的 `ThreadLocal` / `ThreadLocalMap` 源码看。

# 总览

一句话记住实现：**数据不在 ThreadLocal 里，而在每个 Thread 自己的 ThreadLocalMap 里；ThreadLocal 只是查这张表的 key。**

![img_1.png](threadLocal.png)

存储结构是三层：`Thread → ThreadLocalMap → Entry[]`。

# 存储结构

每个线程持有一个 `ThreadLocalMap`：

```java
// Thread 里就这个字段
ThreadLocal.ThreadLocalMap threadLocals = null;
```

Map 里存的是 `Entry`，注意它的 key 是**弱引用**、value 是**强引用**：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;                          // value 是强引用
    Entry(ThreadLocal<?> k, Object v) {
        super(k);                          // key（ThreadLocal 实例）是弱引用
        value = v;
    }
}
```

# set / get / remove

`set` 先拿到当前线程的 Map，再以 `this`（ThreadLocal 实例本身）为 key 塞进去：

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);        // 取当前线程的 ThreadLocalMap
    if (map != null)
        map.set(this, value);              // key 就是当前 ThreadLocal 实例
    else
        createMap(t, value);
}
```

`get` 反过来，用 `this` 去查：

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();              // 没设置过，用 initialValue() 兜底
}
```

轮到 `ThreadLocalMap` 自己了。它**不是链表，而是开放地址法（线性探测）**：

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    // 定位：threadLocalHashCode & (len - 1)，和 HashMap 一个套路
    int i = key.threadLocalHashCode & (len - 1);
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        if (k == key) {                        // 命中，覆盖
            e.value = value;
            return;
        }
        if (k == null) {                       // 撞到 stale 槽位，顺手清理并插入
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)   // 清理不动且到阈值，扩容
        rehash();
}
```

两个细节：

- **hash 递增**：每个 ThreadLocal 的 `threadLocalHashCode` 由 `nextHashCode` 每次加 `0x61c88647`（黄金分割 / Fibonacci 散列）得到，让 key 分布更均匀；
- **扩容阈值约为数组长度的 2/3**，扩容时会顺带清理 key 为 null 的 stale entry。

`remove()` 则是把 Entry 断开、清理 stale 槽位，这也是它比"不调"更重要的原因（下面细说）。

# 为什么 key 是弱引用，value 却是强引用

- **key 弱引用**：ThreadLocal 实例在外部不再被引用时可以被回收，避免它被线程的 Map 一直强引用着；
- **value 强引用**：正常使用时得能取到值。

代价是：**key 被回收后，Entry 变成 `key = null` 的 stale entry，但 value 还挂在 `Thread → ThreadLocalMap → Entry → value` 这条强引用链上**。

# 内存泄漏到底是怎么发生的

先把前提说清楚：内存泄漏是"已分配的内存无法释放"。ThreadLocal 泄漏的典型场景**不是线程结束，恰恰是线程长期存活**（比如线程池里的工作线程）：

1. ThreadLocal 实例被回收后，Entry 的 key 变成 `null`；
2. 但强引用链还在：`Thread → ThreadLocalMap → Entry → value`；
3. 只要线程不死，value 就无法被 GC，直到后续 `get` / `set` / `remove` 触发 `expungeStaleEntry`，或扩容时才被顺带清理。

反过来，**线程真正结束时，`threadLocals` 整体不可达，反而会被正常回收**。

所以规范做法就是：**用完调用 `remove()`**，线程池场景尤其重要。

# InheritableThreadLocal 与 TTL

- `InheritableThreadLocal` 允许子线程在**创建时**继承父线程的值；
- 坑是：线程池里的线程是复用的、不会重新创建，所以子线程拿不到任务提交线程的值；
- 解决：手动传递、装饰 Runnable，或用阿里开源的 `TransmittableThreadLocal`（TTL）。

# 实战场景

## 场景一：登录用户 / 租户上下文

在 Filter 或拦截器里解析登录态，之后 Controller、Service、DAO 任意层都能直接拿到当前用户，不用层层传参：

```java
public final class UserContext {
    private static final ThreadLocal<LoginUser> HOLDER = new ThreadLocal<>();
    public static void set(LoginUser user) { HOLDER.set(user); }
    public static LoginUser get() { return HOLDER.get(); }
    public static void clear() { HOLDER.remove(); }
}
```

```java
try {
    UserContext.set(parseToken(request));
    chain.doFilter(request, response);
} finally {
    UserContext.clear();   // 必须清理
}
```

多租户系统同理：把 `tenantId` 放进 ThreadLocal，DAO / 数据源层统一取值。

## 场景二：链路追踪 traceId（MDC）

日志框架的 MDC 底层就是一个 `ThreadLocal<Map<String, String>>`：

```java
String traceId = UUID.randomUUID().toString().replace("-", "");
MDC.put("traceId", traceId);
try {
    // 业务逻辑
} finally {
    MDC.clear();
}
```

logback 的 pattern 里用 `%X{traceId}` 输出。注意：异步线程 / 线程池不会自动传递 MDC。

## 场景三：事务与数据库连接

Spring 的 `TransactionSynchronizationManager` 用 ThreadLocal 保存当前线程绑定的 `Connection`、事务状态、同步器等，所以同一线程内的多个 DAO 能共享同一个事务连接。MyBatis 类似：`SqlSessionUtils` 把 `SqlSessionHolder` 放进 ThreadLocal，保证同一事务里用同一个 SqlSession。

这也是为什么**事务方法里新开线程，事务会失效**——新线程的 ThreadLocal 是空的，拿不到原线程的连接。

## 场景四：线程不安全的工具类

`SimpleDateFormat` 内部的 `Calendar` 不是线程安全的，多线程共享会解析出错。每个线程一份即可：

```java
private static final ThreadLocal<SimpleDateFormat> DATE_FORMAT =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
```

同类问题还有 `DecimalFormat` 等。

## 场景五：动态数据源 / 分库分表路由

AOP 在方法执行前把数据源标识放进 ThreadLocal，`AbstractRoutingDataSource.determineCurrentLookupKey()` 从 ThreadLocal 读，决定用哪个库：

```java
@Around("@annotation(ds)")
public Object route(ProceedingJoinPoint pjp, DS ds) throws Throwable {
    DynamicDataSourceHolder.set(ds.value());
    try {
        return pjp.proceed();
    } finally {
        DynamicDataSourceHolder.clear();
    }
}
```

## 场景六：框架里的 ThreadLocal

- Spring `RequestContextHolder`：用 `NamedThreadLocal` 存 `RequestAttributes`；
- `LocaleContextHolder`：存当前线程的 Locale；
- Logback / Log4j2 的 MDC；
- Spring 事务同步、MyBatis SqlSession 管理等。

可以说：**"同一线程内跨层共享状态"的地方，基本都是 ThreadLocal。**

# 常见坑

1. **忘记 remove**：线程池复用时读到上一个任务的数据（最严重）；
2. **异步丢失上下文**：`@Async`、`CompletableFuture`、自定义线程池都不会继承普通 ThreadLocal；
3. **InheritableThreadLocal 在池化线程里失效**：线程只创建一次，之后不再继承；
4. **存大对象**：value 是强引用，线程不死就一直不释放；
5. **`get()` 返回 null 有歧义**：可能是没设置，也可能是设置成了 null；需要区分时用 `containsKey` 或约定不存 null。

# 小结

1. **ThreadLocal 和 synchronized 的区别？** synchronized 是时间换空间，多线程串行访问同一份共享变量；ThreadLocal 是空间换时间，每个线程一份副本、没有竞争。前者解决共享变量的并发安全，后者解决"线程私有状态"的隔离与传递。
2. **数据存在哪？** 不在 ThreadLocal 里，而在每个 Thread 的 `ThreadLocalMap` 里，ThreadLocal 只是 key。
3. **为什么 key 弱引用、value 强引用？** key 弱引用让 ThreadLocal 实例可被回收；value 强引用保证取值。代价是易产生 `key=null` 的 stale entry。
4. **内存泄漏怎么产生？** 线程长期存活时，key 被回收但 value 仍被 `Thread → Map → Entry → value` 强引用挂着，直到 get/set/remove 清理。注意是"线程长期存活"而非"线程结束"。
5. **为什么推荐 `static final`？** 一个 ThreadLocal 实例对应 Map 里一个 entry，反复 new 会产生大量 entry 且易忘 remove；声明成 `static final` 全类共用一份。
6. **InheritableThreadLocal 有什么坑？** 子线程只在创建时继承；线程池线程复用、不再创建，所以拿不到提交线程的值，需要用 TTL。
7. **线程池里为什么必须 remove？** 线程复用，上个任务没清理的值会被下个任务读到，造成串号、越权、日志错乱。写法必须是 `try { ... } finally { remove(); }`。

# 参考

- JDK8 `ThreadLocal`、`ThreadLocalMap` 源码
- 阿里 `TransmittableThreadLocal`（TTL）
