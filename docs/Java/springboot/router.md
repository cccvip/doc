SaaS 多租户、业务增长后的分库分表，都会遇到同一个问题：一次请求该连哪个库、查哪张表？如果每个 DAO 方法都手动传 `tenantId`、手动拼表名，代码会到处是重复逻辑，还容易漏。

这篇用一个"垂直分库"的例子，把「AOP + ThreadLocal + AbstractRoutingDataSource」这套动态数据源方案走一遍，并列出实际落地时最容易踩的坑。自动装配和 Starter 的写法见 [starter.md](starter.md)。

# 总览

**一句话：AOP 在方法执行前把"用哪个库"放进 ThreadLocal，`AbstractRoutingDataSource` 在执行 SQL 前从 ThreadLocal 取出 key，路由到对应的数据源。**

```text
请求进入
  → @DatabaseSharding("tenantId") 标注的方法
      → DynamicDataSourceAspect（@Around）
          → 从参数里取 tenantId
          → DynamicDataSourceHolder.set(tenantId)
          → 执行业务
          → finally 清理 ThreadLocal

执行 SQL
  → AbstractRoutingDataSource.determineCurrentLookupKey()
      → DynamicDataSourceHolder.get()
      → 从 targetDataSources 里选对应的 DataSource
```

# 核心概念

## 为什么要分库分表

单库单表在数据量、并发量上来之后会遇到瓶颈：

- 单表数据量大，索引变深，查询变慢；
- 单库连接数、磁盘 IO、CPU 到顶；
- 备份、DDL、扩容都变得困难。

分库分表就是把数据拆到多个库 / 多张表，降低单点压力。

## 垂直拆分 vs 水平拆分

| 方式 | 含义 | 例子 |
|---|---|---|
| 垂直拆分 | 按业务 / 租户拆库 | 每个租户一个库，或订单库、用户库分离 |
| 水平拆分 | 同一张表按行拆成多张 | `order_001`、`order_002`、`order_003` |

这篇做的是**垂直拆分（多租户分库）**：每个租户一个库，按 `tenantId` 路由。分表暂不做，原因是短期内数据量还到不了瓶颈。

## AbstractRoutingDataSource 原理

Spring 提供的 `AbstractRoutingDataSource` 本身就是一个 `DataSource`，内部维护一组目标数据源：

```java
public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean {

    private Map<Object, Object> targetDataSources;
    private Object defaultTargetDataSource;

    @Override
    public Connection getConnection() throws SQLException {
        return determineTargetDataSource().getConnection();
    }

    protected DataSource determineTargetDataSource() {
        Object lookupKey = determineCurrentLookupKey();
        DataSource dataSource = this.targetDataSources.get(lookupKey);
        return (dataSource != null ? dataSource : this.defaultTargetDataSource);
    }

    // 子类实现：返回当前线程要用哪个数据源
    @Nullable
    protected abstract Object determineCurrentLookupKey();
}
```

关键就是 `determineCurrentLookupKey()`：每次获取连接时，它决定用哪个目标数据源。

## 为什么用 ThreadLocal

数据源的选择发生在 DAO 层，而租户信息通常来自 Controller 层的参数或登录态。用 ThreadLocal 把它从上层传到下层，避免每个方法都显式传参。

```java
public final class DynamicDataSourceHolder {
    private static final ThreadLocal<String> DS = new ThreadLocal<>();

    public static void set(String ds) {
        DS.set(ds);
    }

    public static String get() {
        return DS.get();
    }

    public static void clear() {
        DS.remove();
    }
}
```

# 实现

## 1. 数据源配置

```java
@Configuration
@AutoConfigureBefore(DataSourceAutoConfiguration.class)
public class DynamicDataSourceConfiguration {

    @Bean
    @ConfigurationProperties("spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.tenant1")
    public DataSource tenant1DataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @Primary
    public DataSource dataSource(DataSource masterDataSource, DataSource tenant1DataSource) {
        Map<Object, Object> targets = new HashMap<>();
        targets.put("master", masterDataSource);
        targets.put("tenant1", tenant1DataSource);

        DynamicRoutingDataSource routing = new DynamicRoutingDataSource();
        routing.setTargetDataSources(targets);
        routing.setDefaultTargetDataSource(masterDataSource);
        return routing;
    }
}
```

`DynamicRoutingDataSource` 只需要实现 `determineCurrentLookupKey`：

```java
public class DynamicRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return DynamicDataSourceHolder.get();
    }
}
```

`@Primary` 很重要：容器里有多个 `DataSource` 时，MyBatis / JdbcTemplate 默认注入的就是这个路由数据源。

## 2. AOP 切换数据源

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface DatabaseSharding {
    String value();   // 用哪个参数作为分库键
}
```

```java
@Aspect
@Component
@Order(0)   // 必须比事务拦截器先执行
public class DynamicDataSourceAspect {

    @Around("@annotation(databaseSharding)")
    public Object route(ProceedingJoinPoint pjp, DatabaseSharding databaseSharding) throws Throwable {
        MethodSignature signature = (MethodSignature) pjp.getSignature();
        String[] paramNames = signature.getParameterNames();
        Object[] args = pjp.getArgs();

        String shardingValue = null;
        for (int i = 0; i < paramNames.length; i++) {
            if (databaseSharding.value().equals(paramNames[i])) {
                shardingValue = String.valueOf(args[i]);
                break;
            }
        }
        if (shardingValue == null) {
            throw new IllegalArgumentException("@DatabaseSharding 指定的参数不存在: " + databaseSharding.value());
        }

        DynamicDataSourceHolder.set(shardingValue);
        try {
            return pjp.proceed();
        } finally {
            DynamicDataSourceHolder.clear();   // 必须清理，否则线程池复用会串库
        }
    }
}
```

`@Order(0)` 的作用是让数据源切面先于事务切面执行，保证事务开启时拿到的连接来自正确的数据源。

## 3. MyBatis 拦截器：替换表名（可选）

如果要做水平分表，可以在 MyBatis 的 `StatementHandler.prepare` 阶段改 SQL：

```java
@Intercepts(@Signature(type = StatementHandler.class, method = "prepare",
        args = {Connection.class, Integer.class}))
public class TableShardingInterceptor implements Interceptor {

    private static final Pattern TABLE_PATTERN =
            Pattern.compile("\\b(from|into|update)\\s+([a-zA-Z0-9_]+)", Pattern.CASE_INSENSITIVE);

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler handler = (StatementHandler) invocation.getTarget();
        MetaObject metaObject = SystemMetaObject.forObject(handler);
        MappedStatement ms = (MappedStatement) metaObject.getValue("delegate.mappedStatement");

        // 只有标了 @DatabaseSharding 的 Mapper 才分表
        String mapperId = ms.getId();
        Class<?> mapperClass = Class.forName(mapperId.substring(0, mapperId.lastIndexOf('.')));
        if (!mapperClass.isAnnotationPresent(DatabaseSharding.class)) {
            return invocation.proceed();
        }

        BoundSql boundSql = handler.getBoundSql();
        String shard = DynamicDataSourceHolder.get();
        if (shard == null || shard.isEmpty()) {
            return invocation.proceed();
        }

        Matcher matcher = TABLE_PATTERN.matcher(boundSql.getSql());
        StringBuffer newSql = new StringBuffer();
        while (matcher.find()) {
            String prefix = matcher.group(1);   // from / into / update
            String table = matcher.group(2);
            matcher.appendReplacement(newSql, prefix + " " + table + "_" + shard);
        }
        matcher.appendTail(newSql);

        // 通过 MetaObject 修改 BoundSql 里的 sql
        metaObject.setValue("delegate.boundSql.sql", newSql.toString());
        return invocation.proceed();
    }
}
```

几个细节：

- 用 `appendReplacement` 逐段替换，不要用 `replaceAll` 直接替换整个匹配；
- 正则只匹配 `from / into / update` 后面的表名，避免误伤字段名；
- 表名、分片值都要做白名单校验，别把外部输入直接拼进 SQL。

## 4. 注册成 Starter

把自动配置类写进清单文件：

- Spring Boot 2.6 及以前：`META-INF/spring.factories`
- Spring Boot 2.7 / 3.x：`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

```properties
# 2.x spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.ds.DynamicDataSourceConfiguration
```

```text
# 2.7+ AutoConfiguration.imports
com.example.ds.DynamicDataSourceConfiguration
```

自动配置类的写法、`@ConditionalOnClass` / `@ConditionalOnMissingBean` 等规范，见 [starter.md](starter.md)。

# 实战与常见坑

## 场景：SaaS 多租户分库

1. 登录后从 token / session 里拿到 `tenantId`；
2. 在 Service 入口方法上标 `@DatabaseSharding("tenantId")`；
3. AOP 把 `tenantId` 放进 ThreadLocal；
4. MyBatis 或 JdbcTemplate 获取连接时，`AbstractRoutingDataSource` 按 `tenantId` 选库；
5. 请求结束，`finally` 清理 ThreadLocal。

## 常见坑

1. **ThreadLocal 不清理**：线程池复用时，下一个请求会读到上一个租户的 `tenantId`，造成**串库**。必须 `finally clear()`。
2. **嵌套切换直接 `remove()`**：内层方法切换数据源后，`remove()` 会把外层的也清掉。应该保存旧值，执行完恢复：
   ```java
   String old = DynamicDataSourceHolder.get();
   try {
       DynamicDataSourceHolder.set(newDs);
       // ...
   } finally {
       if (old == null) DynamicDataSourceHolder.clear();
       else DynamicDataSourceHolder.set(old);
   }
   ```
3. **事务先于数据源切换**：`@Transactional` 在开启事务时就会获取连接；如果数据源切面排在事务之后，切库不会生效。用 `@Order` 保证数据源切面先执行。
4. **同类自调用**：`this.method()` 不走代理，AOP 不生效，数据源不会切换；和 `@Transactional` 自调用失效是同一个原因。
5. **注解拿错位置**：`joinPoint.getTarget().getClass().getAnnotation(...)` 只能拿到类上的注解，拿不到方法上的；要用 `MethodSignature.getMethod()` 或 `AnnotatedElementUtils.findMergedAnnotation`。
6. **`getParameterNames()` 返回 `arg0`**：编译时没加 `-parameters`，参数名拿不到；要么加编译参数，要么用注解显式指定参数下标。
7. **SQL 替换误伤**：`replaceAll` 会把整个 `from table` 替换掉，丢掉 `from`；用 `appendReplacement`，并且只匹配目标表。
8. **`assert` 不生效**：JVM 默认关闭断言，`assert tableName != null` 等于没写；该抛异常就抛异常。
9. **多数据源事务管理器选错**：每个数据源要配自己的 `DataSourceTransactionManager`，`@Transactional` 要指定对应的事务管理器。
10. **跨库事务**：`@Transactional` 只对当前数据源生效；跨库要么避免，要么上 Seata 这类分布式事务方案。

# 小结

我一向不太喜欢写总结，因为该说的正文里都说了。不过考虑到很多读者是冲着面试来的，还是列成清单：

1. **动态数据源的原理？** AOP 把数据源标识放进 ThreadLocal，`AbstractRoutingDataSource.determineCurrentLookupKey()` 取出并选择目标数据源。
2. **为什么用 ThreadLocal？** 把租户信息从 Controller 传到 DAO，避免层层传参；用完必须清理。
3. **为什么 AOP 要排在事务前面？** 事务开启时就会拿连接，数据源切换必须在拿连接之前。
4. **垂直拆分和水平拆分的区别？** 垂直按业务 / 租户拆库，水平按行拆表。
5. **分表 SQL 怎么改？** MyBatis 拦截 `StatementHandler.prepare`，用 `appendReplacement` 替换表名。
6. **最大的坑是什么？** ThreadLocal 不清理导致串库；其次是事务与切换顺序。
7. **怎么注册成 Starter？** 自动配置类写进 `spring.factories` 或 `AutoConfiguration.imports`，规范见 [starter.md](starter.md)。

# 参考

- Spring `AbstractRoutingDataSource` 源码
- MyBatis `Interceptor` / `StatementHandler` 源码
- [starter.md](starter.md)（自动装配与 Starter）、[../spring/SpringAop代理.md](../spring/SpringAop代理.md)
