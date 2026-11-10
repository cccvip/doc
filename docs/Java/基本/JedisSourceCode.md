`JedisPool` 看着简单，但"底层是什么""为什么必须 close""连接耗尽怎么回事"这些问题，翻一下配置文档是答不全的。这篇从 commons-pool2 的 `borrowObject` / `returnObject` 走一遍，把借还连接的机制讲清楚。

> 本文对着 apache-commons-pool2 的 `GenericObjectPool` 和 Jedis 的 `JedisPool` / `JedisFactory` 源码看。

# 总览

Jedis 的连接池不是自己实现的，而是基于 **apache-commons-pool2** 的 `GenericObjectPool`。`JedisPool` 只负责把「创建、销毁、校验 Jedis」的细节交给 `JedisFactory`。

一句话：**JedisPool = GenericObjectPool\<Jedis\> + JedisFactory。**

# 核心原理

## 三个对象

- `JedisPool`：对外入口，`getResource()` 借连接；
- `GenericObjectPool<Jedis>`：池的核心，管理空闲队列、最大连接数、等待、驱逐；
- `JedisFactory`：模板模式，实现 `create()` / `destroy()` / `validateObject()`，真正 new Jedis、close、PING。

```java
public abstract class Pool<T> implements Closeable {
    public T getResource() {
        return internalPool.borrowObject();   // 借连接，交给底层的 GenericObjectPool
    }
}
```

## borrowObject 流程

1. 先从空闲队列取（默认 LIFO，取最近归还的，缓存更热）；
2. 没有空闲且当前总数 < `maxTotal`：`factory.create()` 新建；
3. 达到 `maxTotal` 且没有空闲：按 `maxWaitMillis` 阻塞等待；
4. 等不到：抛 `NoSuchElementException`，Jedis 包装成 `JedisExhaustedPoolException`；
5. 如果 `testOnBorrow=true`，取到后先 `validateObject()`，失败就销毁并重试。

![onBorrow](_assets/onBorrow.png)

## returnObject 流程

`Jedis.close()` 不是关闭 TCP，而是把连接还给池：

```java
public void close() {
    if (dataSource != null) {
        JedisPoolAbstract pool = this.dataSource;
        this.dataSource = null;
        if (isBroken()) {
            pool.returnBrokenResource(this);   // 标记为坏连接 → 直接销毁
        } else {
            pool.returnResource(this);         // 正常归还到空闲队列
        }
    } else {
        super.close();                          // 不是从池里借的，才真正关闭
    }
}
```

- `returnResource`：正常归还；
- `returnBrokenResource`：标记为坏连接，直接销毁；
- 归还时如果 `testOnReturn=true`，先 `validateObject()`，失败则销毁并补一个空闲连接；
- 空闲数超过 `maxIdle` 时，多出来的连接会被销毁。

```java
// 归还前校验失败：销毁这个连接，并补一个空闲连接维持 minIdle
if (getTestOnReturn() && !factory.validateObject(p)) {
    destroy(p, DestroyMode.NORMAL);
    ensureIdle(1, false);
    return;
}
```

## validateObject 做了什么

`JedisFactory.validateObject()` 本质是发一次 `PING`，能正常收到 `PONG` 就认为连接可用。所以 `testOnBorrow` / `testOnReturn` 每开一个，就多一次 RTT。

## 空闲检测

`testWhileIdle` + `timeBetweenEvictionRunsMillis`：后台 Evictor 定期扫描空闲连接，验证并驱逐超过 `minEvictableIdleTimeMillis` 的连接。

适合网络不稳定、或者 Redis 配置了 `timeout` 主动断开空闲连接的场景，否则容易拿到已被服务端关掉的死连接。

# 配置参数

## 容量三兄弟

| 参数 | 含义 | 建议 |
|---|---|---|
| `maxTotal` | 最大连接数 | ≈ QPS × RTT；且 nodes × maxTotal < Redis `maxclients` |
| `maxIdle` | 最大空闲连接数 | 建议 = maxTotal，避免频繁销毁/创建 |
| `minIdle` | 最小空闲连接数 | 预热用，取 maxIdle 的 1/10 左右 |
| `maxWaitMillis` | 借连接最长等待 | 不要用 -1（永久阻塞），设 1~3s |

估算：RTT 1ms 时单连接约 1000 QPS，目标 50000 QPS → 约 50 个连接。

注意：**Redis 执行命令是单线程的**，连接数不是越多越好，连接过多反而增加上下文切换和 Redis 开销。

## 校验三兄弟

| 参数 | 时机 | 代价 |
|---|---|---|
| `testOnBorrow` | 借出前校验 | 每次借连接多一次 PING |
| `testOnReturn` | 归还前校验 | 每次归还多一次 PING |
| `testWhileIdle` | 后台定期校验空闲连接 | 由 Evictor 承担，推荐 |

高频场景一般 `testWhileIdle=true` + `testOnBorrow=false`；网络差再考虑开 `testOnBorrow`。

## 其他常用

`blockWhenExhausted`、`lifo`、`fairness`、`timeBetweenEvictionRunsMillis`、`minEvictableIdleTimeMillis`、`numTestsPerEvictionRun`。

# 实战场景

## Spring Boot 配置

```yaml
# Spring Boot 2.x
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    timeout: 2000ms
    jedis:
      pool:
        max-active: 50
        max-idle: 50
        min-idle: 5
        max-wait: 2000ms
```

Spring Boot 3.x 前缀改为 `spring.data.redis`。

## 正确使用

```java
try (Jedis jedis = jedisPool.getResource()) {
    jedis.set("k", "v");
}
```

`try-with-resources` 能保证异常时也归还连接。

## 连接耗尽排查

现象：

```text
JedisExhaustedPoolException: Could not get a resource from the pool
```

按顺序查：

1. 是否有没 `close` 的代码路径；
2. 是否有阻塞命令 / 慢命令长时间占连接；
3. `maxTotal` 是否太小；
4. Redis 是否变慢（`slowlog`、`latency`）；
5. 服务端连接数：`redis-cli info clients`、`client list`。

## 集群模式

`JedisCluster` 内部按节点维护连接池。估算总连接时要乘节点数，并确认每个 Redis 节点的 `maxclients` 够用。

# 常见坑

1. **忘记 close**：连接泄漏，最终池耗尽；
2. **close 不在 finally**：异常路径漏还连接；
3. **maxWaitMillis = -1**：故障时线程永久阻塞；
4. **testOnBorrow 全开**：每次借连接多一次 PING，高 QPS 下开销明显；
5. **maxIdle 远小于 maxTotal**：连接反复销毁、重建；
6. **共享 Jedis 实例**：线程不安全，协议会串；
7. **阻塞命令占连接**：`BLPOP` 不返回，连接一直不还；
8. **忽略 Redis `timeout`**：服务端断开空闲连接，客户端拿到死连接，需要 `testWhileIdle`。

# 小结

1. **JedisPool 底层是什么？** commons-pool2 的 `GenericObjectPool<Jedis>`，配合 `JedisFactory`（`BasePooledObjectFactory<Jedis>`）完成创建、销毁、校验。
2. **maxTotal / maxIdle / minIdle 的区别？** `maxTotal` 是池里最多多少个连接（活跃 + 空闲）；`maxIdle` 是最多保留多少空闲连接，超出的归还时销毁；`minIdle` 是最少保留多少空闲连接，不够会补。一般满足 `maxTotal >= maxIdle >= minIdle`。
3. **testOnBorrow / testOnReturn / testWhileIdle 的区别？** 前两个分别在校验借出前 / 归还前，是同步开销；第三个由后台线程定期扫描空闲连接，是异步检测。
4. **连接耗尽会怎样？** `getResource()` 在 `maxWaitMillis` 内拿不到连接，抛 `JedisExhaustedPoolException`（底层是 `NoSuchElementException`）。
5. **为什么必须 close？** `close()` 把连接还给池。不 close 就等于连接一直被占用，最终池耗尽。
6. **Jedis 实例线程安全吗？** **不安全**。Jedis 内部是一条 TCP 连接 + 请求/响应流，多线程共享会串包。正确做法是每次从 `JedisPool` 借一个，用完还回去。
7. **JedisPool 和 JedisCluster 的区别？** 前者是单节点 / 主从；后者是集群模式，**每个节点一个池**，总连接数 = 节点数 × 单池 `maxTotal`。
8. **连接数怎么估？** `连接数 ≈ QPS × 平均 RTT（秒）`，再留冗余；但要记住 Redis 单线程执行命令，池不是越大越好。
9. **maxWaitMillis = -1 有什么风险？** 拿不到连接时**永久阻塞**，线程会一直挂住，故障时容易把业务线程池拖死。
10. **为什么 maxIdle 建议等于 maxTotal？** `maxIdle < maxTotal` 时，流量高峰后多出来的空闲连接会被销毁，下次高峰又要重新建连，反复创建/销毁不划算。
11. **阻塞命令有什么坑？** `BLPOP` / `BRPOP` 会一直占着连接，不能和普通命令共用一个小池，建议单独池或单独线程。

# 参考

- apache-commons-pool2 `GenericObjectPool`
- Jedis 源码：`JedisPool`、`JedisFactory`
