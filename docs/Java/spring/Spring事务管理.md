`@Transactional` 用起来就一个注解，背后却是一整套"没写一行代码，事务却自动开了又提交"的机制。很多人知道它依赖 AOP，但说不清方法返回和抛异常时分别走到哪个分支、传播行为到底改了什么。

这篇从 `@EnableTransactionManagement` 和 `invokeWithinTransaction` 的源码出发，把"事务在哪开、在哪提交/回滚、ThreadLocal 什么时候清理"走一遍。如果只为面试，可以直接滑到最后的**小结**。

> 基于 SpringBoot 2.5.7。事务的织入依赖 AOP 代理，代理机制见 [SpringAop代理.md](SpringAop代理.md)。

# 总览

**一句话：`@Transactional` 靠 AOP 拦截器 + 事务管理器——`TransactionInterceptor` 在方法执行前开启事务，正常返回时提交，抛异常时回滚。**

```text
@EnableTransactionManagement
  → @Import(AutoProxyRegistrar, ProxyTransactionManagementConfiguration)
      → 注册 AOP 基础设施 + TransactionInterceptor
          → 匹配到 @Transactional 方法
              → TransactionInterceptor.invoke
                  → invokeWithinTransaction
                      ├─ createTransactionIfNecessary  → getTransaction 开启事务
                      ├─ proceedWithInvocation         → 执行业务方法
                      ├─ commitTransactionAfterReturning  正常返回：提交
                      └─ completeTransactionAfterThrowing 抛异常：回滚
                      finally cleanupTransactionInfo     清理 ThreadLocal
```

# 核心概念

## 三个核心对象

| 对象 | 作用 |
|---|---|
| `PlatformTransactionManager` | 事务管理器，负责 `getTransaction` / `commit` / `rollback`；常见实现是 `DataSourceTransactionManager` |
| `TransactionDefinition` | 事务定义：传播行为、隔离级别、超时、是否只读 |
| `TransactionStatus` | 事务状态：是否新事务、是否 rollback-only、是否有 savepoint |

## @Transactional 的常用属性

| 属性 | 默认值 | 说明 |
|---|---|---|
| `propagation` | `REQUIRED` | 传播行为，决定有事务时加入还是新建 |
| `isolation` | `DEFAULT` | 隔离级别，默认跟随数据库 |
| `rollbackFor` | 空 | 默认只对 `RuntimeException` / `Error` 回滚 |
| `readOnly` | `false` | 只读事务，可做优化 |
| `timeout` | `-1` | 超时时间（秒），-1 表示不限制 |

## 七种传播行为

| 传播行为 | 有事务时 | 无事务时 |
|---|---|---|
| `REQUIRED`（默认） | 加入当前事务 | 新建事务 |
| `REQUIRES_NEW` | 挂起当前事务，新建 | 新建事务 |
| `NESTED` | 用 savepoint 嵌套 | 新建事务 |
| `SUPPORTS` | 加入当前事务 | 非事务执行 |
| `NOT_SUPPORTED` | 挂起当前事务 | 非事务执行 |
| `MANDATORY` | 加入当前事务 | 抛异常 |
| `NEVER` | 抛异常 | 非事务执行 |

# 源码分析

## 入口：@EnableTransactionManagement

它通过 `@Import` 引入两个东西：

1. **`AutoProxyRegistrar`**：注册 `InfrastructureAdvisorAutoProxyCreator`，让带 `@Transactional` 的 Bean 被代理；
2. **`ProxyTransactionManagementConfiguration`**：配置事务拦截器 `TransactionInterceptor`、`TransactionAttributeSource` 和 `BeanFactoryTransactionAttributeSourceAdvisor`。

## 哪些方法会被增强：TransactionAttributeSourcePointcut

判断逻辑在 `TransactionAttributeSourcePointcut`：**方法（或类）上能找到事务属性，就匹配**。

```java
abstract class TransactionAttributeSourcePointcut extends StaticMethodMatcherPointcut implements Serializable {
    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        TransactionAttributeSource tas = getTransactionAttributeSource();
        // 能解析出事务属性（即带 @Transactional）就匹配
        return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
    }
}
```

## 拦截器：TransactionInterceptor.invoke

匹配到方法后，由 `TransactionInterceptor` 负责增强：

```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {
    @Override
    @Nullable
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

        // 进入事务处理的真正逻辑
        return invokeWithinTransaction(invocation.getMethod(), targetClass, new CoroutinesInvocationCallback() {
            @Override
            public Object proceedWithInvocation() throws Throwable {
                return invocation.proceed();          // 调下一个拦截器 / 目标方法
            }
            @Override
            public Object getTarget() {
                return invocation.getThis();
            }
            @Override
            public Object[] getArguments() {
                return invocation.getArguments();
            }
        });
    }
}
```

## invokeWithinTransaction 全流程

这是事务的核心，去掉协程和响应式分支后，标准流程如下：

```java
@Nullable
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
        final InvocationCallback invocation) throws Throwable {

    // ① 拿到事务属性（@Transactional 的配置）
    TransactionAttributeSource tas = getTransactionAttributeSource();
    final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);

    // ② 选择事务管理器
    final PlatformTransactionManager tm = determineTransactionManager(txAttr);
    final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

    if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
        // ③ 创建事务，并绑定到当前线程的 ThreadLocal
        TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);

        Object retVal;
        try {
            // ④ 执行业务方法
            retVal = invocation.proceedWithInvocation();
        } catch (Throwable ex) {
            // ⑤ 抛异常：按 rollbackOn 判断回滚还是提交
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        } finally {
            // ⑥ 清理 ThreadLocal：恢复外层事务信息
            cleanupTransactionInfo(txInfo);
        }

        // ⑦ 正常返回：提交
        commitTransactionAfterReturning(txInfo);
        return retVal;
    } else {
        // CallbackPreferringPlatformTransactionManager 分支，略
        // ...
    }
}
```

四个关键动作：

- **创建事务**：`createTransactionIfNecessary` → `tm.getTransaction(txAttr)`；
- **执行业务**：`invocation.proceedWithInvocation()`，也就是 AOP 链继续往下走；
- **提交或回滚**：正常返回走 `commitTransactionAfterReturning`，抛异常走 `completeTransactionAfterThrowing`；
- **清理**：`cleanupTransactionInfo`，恢复外层事务信息，防止嵌套调用后 ThreadLocal 错乱。

## createTransactionIfNecessary：创建事务并绑定 ThreadLocal

```java
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
        @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

    // 事务没有名字时，用方法签名作为名字（日志里能看出来是哪个方法）
    if (txAttr != null && txAttr.getName() == null) {
        txAttr = new DelegatingTransactionAttribute(txAttr) {
            @Override
            public String getName() {
                return joinpointIdentification;
            }
        };
    }

    TransactionStatus status = null;
    if (txAttr != null) {
        if (tm != null) {
            // 真正开启事务：getTransaction
            status = tm.getTransaction(txAttr);
        }
    }

    // 把事务信息绑定到当前线程
    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

`prepareTransactionInfo` 创建 `TransactionInfo` 并调用 `bindToThread`：

```java
private void bindToThread() {
    // 记下旧的 TransactionInfo，再把新的放进去（支持事务嵌套）
    this.oldTransactionInfo = transactionInfoHolder.get();
    transactionInfoHolder.set(this);
}
```

`transactionInfoHolder` 是 `TransactionAspectSupport` 里的 `ThreadLocal<TransactionInfo>`，所以事务信息只在当前线程有效。

## getTransaction：事务管理器怎么开事务

`getTransaction` 在 `AbstractPlatformTransactionManager` 里，核心是"先看有没有现成事务，再按传播行为处理"：

```java
@Override
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
        throws TransactionException {

    // ① 看当前线程有没有已存在的事务
    Object transaction = doGetTransaction();
    if (definition == null) {
        definition = new DefaultTransactionDefinition();
    }

    // ② 有事务：交给 handleExistingTransaction，按传播行为决定加入 / 挂起 / 嵌套
    if (isExistingTransaction(transaction)) {
        return handleExistingTransaction(definition, transaction, debugEnabled);
    }

    // ③ 没有事务：校验传播行为和超时
    if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
    }

    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
                "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        // ④ 新建事务：挂起当前（如果有），然后 doBegin 真正开启
        SuspendedResourcesHolder suspendedResources = suspend(null);
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            doBegin(transaction, definition);   // 这里才会 setAutoCommit(false)
            prepareSynchronization(status, definition);
            return status;
        } catch (RuntimeException | Error ex) {
            resume(null, suspendedResources);
            throw ex;
        }
    }
    else {
        // SUPPORTS / NOT_SUPPORTED / NEVER：不新建事务
        // ...
    }
}
```

`doBegin` 的具体实现由子类决定，比如 `DataSourceTransactionManager.doBegin` 会从 `DataSource` 取连接、执行 `connection.setAutoCommit(false)`，并把连接绑定到 `TransactionSynchronizationManager`。

## 提交与回滚：commit / rollback

`commitTransactionAfterReturning` 只是把状态交给事务管理器：

```java
protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
    }
}
```

`AbstractPlatformTransactionManager.commit` 的入口：

```java
@Override
public final void commit(TransactionStatus status) throws TransactionException {
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException(
                "Transaction is already completed - do not call commit or rollback more than once per transaction");
    }

    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;

    // 已经被标记 rollback-only，直接回滚
    if (defStatus.isLocalRollbackOnly()) {
        processRollback(defStatus, false);
        return;
    }

    if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
        processRollback(defStatus, true);
        return;
    }

    processCommit(defStatus);
}
```

`processCommit` 里会区分"新事务"和"加入已有事务"：新事务才真正 `doCommit`，加入已有事务则什么都不做，由最外层事务统一提交。

异常分支：

```java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        // rollbackOn 默认只认 RuntimeException / Error
        if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
            try {
                txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
            } catch (TransactionSystemException ex2) {
                // ...
            }
        } else {
            // 不满足回滚规则：仍然提交
            try {
                txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
            } catch (TransactionSystemException ex2) {
                // ...
            }
        }
    }
}
```

`rollbackOn` 的默认实现：

```java
public boolean rollbackOn(Throwable ex) {
    // 默认只对运行时异常和 Error 回滚
    return (ex instanceof RuntimeException || ex instanceof Error);
}
```

这就是"受检异常默认不回滚"的源码依据。

## 为什么必须 public

`computeTransactionAttribute` 会直接跳过非 public 方法：

```java
protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
    // Don't allow no-public methods as required.
    if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
        return null;
    }
    // ...
}
```

# 实战与常见坑

## 场景：自调用与异常不回滚

**自调用**：

```java
@Service
class OrderService {

    public void create() {
        this.pay();          // 自调用：this 是原始对象，不是代理
    }

    @Transactional
    public void pay() {
        // ...
    }
}
```

`this.pay()` 没有经过代理，`TransactionInterceptor` 没有机会介入，事务不会生效。解决方式和 AOP 自调用一致：注入自身代理、`AopContext.currentProxy()`、或拆到另一个 Bean。

**受检异常不回滚**：

```java
@Transactional
public void pay() throws IOException {
    // 抛 IOException：默认不回滚！
    throw new IOException("boom");
}
```

默认 `rollbackOn` 只认 `RuntimeException` / `Error`。要让受检异常也回滚，必须显式指定：

```java
@Transactional(rollbackFor = Exception.class)
```

> `REQUIRES_NEW` 会挂起当前事务、新开一个连接。嵌套层级多时要注意连接池大小，否则容易出现"事务拿了两个连接、池子不够"的等待。

## 常见坑

1. **加在非 public 方法上**：`computeTransactionAttribute` 直接返回 null，事务失效。
2. **同类内部自调用**：`this.method()` 不走代理，拦截器不介入。
3. **受检异常默认不回滚**：需要 `rollbackFor = Exception.class`。
4. **捕获异常后没抛出**：`completeTransactionAfterThrowing` 收不到异常，事务会正常提交；要么别吞，要么手动 `setRollbackOnly()`。
5. **以为事务能跨线程**：事务信息在 `ThreadLocal`，新线程拿不到；异步任务要单独开事务。
6. **`REQUIRES_NEW` 滥用**：每个新事务占一个连接，嵌套多了会耗尽连接池。
7. **多数据源没指定事务管理器**：`@Transactional("orderTxManager")` 或 `@Qualifier` 指定，否则可能用错管理器。
8. **只读事务里写数据**：`readOnly=true` 只是提示，不保证数据库拒绝写，别把它当约束。

# 小结

我一向不太喜欢写总结，因为该说的正文里都说了。不过考虑到很多读者是冲着面试来的，还是列成清单：

1. **`@Transactional` 靠什么实现？** AOP 拦截器（`TransactionInterceptor`）+ `PlatformTransactionManager`。
2. **`@EnableTransactionManagement` 做了什么？** 注册 `InfrastructureAdvisorAutoProxyCreator` 和事务拦截器。
3. **哪些方法会被增强？** `TransactionAttributeSourcePointcut` 能解析出事务属性就匹配。
4. **核心流程？** `createTransactionIfNecessary` → `getTransaction` 开事务 → 执行业务 → 提交/回滚 → 清理 ThreadLocal。
5. **什么时候回滚？** `rollbackOn` 默认只认 `RuntimeException` / `Error`；受检异常要配 `rollbackFor`。
6. **传播行为是什么？** 有事务时是加入还是新建，`REQUIRED` 默认加入，`REQUIRES_NEW` 挂起后新建。
7. **为什么必须 public？** `computeTransactionAttribute` 对非 public 方法直接返回 null。
8. **为什么自调用失效？** `this` 是原始对象，不是代理，拦截器没有机会介入。

# 参考

- [Spring Framework 官方文档：Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- Spring 5.x 源码：`TransactionInterceptor`、`TransactionAspectSupport`、`AbstractPlatformTransactionManager`
- [SpringAop代理.md](SpringAop代理.md)、[Spring依赖注入.md](Spring依赖注入.md)
