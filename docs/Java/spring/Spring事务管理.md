`@Transactional` 用起来就一个注解，背后却是一整套"没写一行代码，事务却自动开了又提交"的机制。这篇从 `@EnableTransactionManagement` 讲起，把事务是怎么被织入、又是在哪一步开启和提交的走一遍。

> 基于 SpringBoot 2.5.7。事务的织入依赖 AOP 代理，代理机制见 [SpringAop代理.md](SpringAop代理.md)。

# 总览

一句话：**`@Transactional` 是靠 AOP 拦截器 + 事务管理器实现的**——给带 `@Transactional` 的方法包一层 `TransactionInterceptor`，在方法执行前后负责开启、提交/回滚事务。

整条链是：

```text
@EnableTransactionManagement
  → @Import(AutoProxyRegistrar, ProxyTransactionManagementConfiguration)
    → 注册 AOP 基础设施 + TransactionInterceptor
      → 匹配到 @Transactional 方法
        → TransactionInterceptor.invoke → invokeWithinTransaction
          → 开启事务 → 执行业务 → 提交/回滚 → 清理 ThreadLocal
```

# 开启事务：@EnableTransactionManagement

它通过 `@Import` 引入两个东西：

1. **`AutoProxyRegistrar`**：生成需要代理的 Bean，注册到 Bean 工厂；
2. **`ProxyTransactionManagementConfiguration`**：事务处理配置类，配置了 AOP 的切入点（凡是带 `@Transactional` 的方法），以及 `TransactionInterceptor`。

# 哪些方法会被加事务

判断逻辑在 `TransactionAttributeSourcePointcut`：**方法（或类）上能找到事务属性，就匹配**。

```java
abstract class TransactionAttributeSourcePointcut extends StaticMethodMatcherPointcut implements Serializable {
    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        TransactionAttributeSource tas = getTransactionAttributeSource();
        return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
    }
}
```

# 拦截器：TransactionInterceptor

匹配到方法后，由 `TransactionInterceptor` 负责增强：

```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {
    @Override
    @Nullable
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
        // 进入事务处理的真正逻辑
        return invokeWithinTransaction(invocation.getMethod(), targetClass, new CoroutinesInvocationCallback() {
            @Override public Object proceedWithInvocation() throws Throwable { return invocation.proceed(); }
            @Override public Object getTarget() { return invocation.getThis(); }
            @Override public Object[] getArguments() { return invocation.getArguments(); }
        });
    }
}
```

# invokeWithinTransaction 的核心步骤

逻辑较长，简化成四步：

1. **创建事务，绑定到 ThreadLocal**：

```java
protected TransactionInfo prepareTransactionInfo(@Nullable PlatformTransactionManager tm,
        @Nullable TransactionAttribute txAttr, String joinpointIdentification,
        @Nullable TransactionStatus status) {
    TransactionInfo txInfo = new TransactionInfo(tm, txAttr, joinpointIdentification);
    // ... 开启事务（doBegin）
    txInfo.bindToThread();
    return txInfo;
}

private void bindToThread() {
    // 记下旧的 TransactionInfo，再把新的放进去（支持事务嵌套）
    this.oldTransactionInfo = transactionInfoHolder.get();
    transactionInfoHolder.set(this);
}
```

2. **`doBegin()`**：真正向数据源要连接，`connection.setAutoCommit(false)`；
3. **执行 `@Transactional` 标注的方法**；
4. **提交或回滚**，然后**清理 ThreadLocal**：

```java
private void restoreThreadLocalStatus() {
    // 把上一个事务的信息放回 ThreadLocal
    transactionInfoHolder.set(this.oldTransactionInfo);
}
```

# 为什么 @Transactional 必须加在 public 方法上

`@Transactional` 只对 public 方法生效，否则会被直接跳过：

```java
public abstract class AbstractFallbackTransactionAttributeSource
        implements TransactionAttributeSource, EmbeddedValueResolverAware {

    protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
        // 非 public 方法不处理
        if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
            return null;
        }
        // ...
    }
}
```

# 常见坑

1. **加在非 public 方法上**：直接失效（见上）；
2. **同类内部自调用**：`this.method()` 不走代理，事务失效——这是 AOP 代理的通病，不是事务本身的问题；
3. **异常类型不对**：默认只对 `RuntimeException` / `Error` 回滚，受检异常默认不回滚，需要 `rollbackFor` 指定；
4. **以为能跨线程**：事务信息在 ThreadLocal 里，新线程拿不到（和 [Spring依赖注入.md](Spring依赖注入.md) 里说的连接绑定是同一个道理）。

# 小结

1. **`@Transactional` 靠什么实现？** AOP 拦截器（`TransactionInterceptor`）+ 事务管理器。
2. **`@EnableTransactionManagement` 做了什么？** 通过 `@Import` 引入 `AutoProxyRegistrar` 和 `ProxyTransactionManagementConfiguration`，注册 AOP 基础设施与事务拦截器。
3. **哪些方法会被增强？** `TransactionAttributeSourcePointcut` 判断：方法/类上能找到事务属性（即带 `@Transactional`）就匹配。
4. **核心执行流程？** 创建事务并绑定 ThreadLocal → `doBegin`（`setAutoCommit(false)`）→ 执行业务 → 提交/回滚 → 恢复 ThreadLocal。
5. **为什么必须 public？** `computeTransactionAttribute` 对非 public 方法直接返回 null。
6. **为什么自调用会失效？** 因为没走代理，拦截器根本没机会介入。

# 参考

- Spring 5.x 源码：`TransactionInterceptor`、`TransactionAspectSupport`、`ProxyTransactionManagementConfiguration`
- [SpringAop代理.md](SpringAop代理.md)、[Spring依赖注入.md](Spring依赖注入.md)
