# 前言
数据库路由组件的意义是帮助应用程序将数据库操作请求路由到正确的数据库实例或分片上。在大型应用程序中，数据库通常会被分布到多个服务器上，而数据库路由组件可以根据一些规则或策略来确定应该将请求发送到哪个数据库实例。

数据库路由组件的主要意义包括：

- 负载均衡：当应用程序需要处理大量的数据库请求时，数据库路由组件可以将请求均匀地分发到多个数据库实例上，以实现负载均衡，提高系统的整体性能和吞吐量。

- 分片管理：数据库分片是一种将数据水平分割存储在多个数据库实例上的技术。数据库路由组件可以根据分片键或哈希算法将数据请求路由到正确的分片上，实现数据的分布式存储和查询。

- 故障容错：当某个数据库实例发生故障时，数据库路由组件可以自动将请求重新路由到其他可用的实例上，确保应用程序的可用性和数据的完整性。

- 多租户支持：在多租户应用程序中，数据库路由组件可以根据不同的租户标识将请求路由到相应的数据库实例上，实现数据的隔离和安全性。

# 保命提示
为什么不用现成的开源的组件？为啥要自己写？ 不要杠,杠就是你对

# 遇到过的场景
- 业务增长过快遇到的分表情况
- Sass服务,多租户存储以及查询

# 需求分析
分库分表常用的两种实现方案,
- 垂直拆分：例如多租户,将创建多个数据库,每一个客户,都需要查询自己的数据
- 水平拆分：水平拆分是将一个表,分成多份的表。例如订单记录,order_001,order_002,order_003。

本次将使用垂直拆分,做一个单纯的分库,满足业务需求。不实现分表的主要原因是(在可见的3年内,不会遇到数据的增长瓶颈)
# 技术方案
1.AOP切面拦截的使用，这是因为需要给使用数据库路由的方法做上标记，便于处理分库分表逻辑 
2.多数据源的切换操作，既然有分库那么就会涉及在多个数据源间进行链接切换，以便把数据分配给不同的数据库,不同的Schema
综上,我们需要用到AOP、数据源切换、SpringBoot-Starter,ThreadLocal等技术


# 技术实现

## 数据源切换

核心类AbstractRoutingDataSource

AbstractRoutingDataSource提供了基本的数据源路由功能，它可以根据当前的上下文或者特定的条件来动态地选择合适的数据源。这个抽象类需要我们自己来实现其中的抽象方法，以便根据自己的业务需求来确定数据源的选择逻辑。

但是在我写下文章的今日,已经有很多开源优秀的产品,实现多数据源

```shell script
    <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
        </dependency>
```

```java
@Configuration
@AutoConfigureBefore({DynamicDataSourceAutoConfiguration.class, SpringBootConfiguration.class})
public class DataSourceConfiguration {
    
    @Resource
    private DynamicDataSourceProperties properties;

    @Bean
    public DynamicDataSourceProvider dynamicDataSourceProvider() {
        Map<String, DataSourceProperty> datasourceMap = properties.getDatasource();
        return new AbstractDataSourceProvider() {
            @Override
            public Map<String, DataSource> loadDataSources() {
                Map<String, DataSource> dataSourceMap = createDataSourceMap(datasourceMap);
                return dataSourceMap;

            }
        };
    }
    
    @Primary
    @Bean(name = "master")
    public DataSource dataSource(DynamicDataSourceProvider dynamicDataSourceProvider) {
        DynamicRoutingDataSource dataSource = new DynamicRoutingDataSource();
        dataSource.setPrimary(properties.getPrimary());
        dataSource.setStrict(properties.getStrict());
        dataSource.setStrategy(properties.getStrategy());
        dataSource.setProvider(dynamicDataSourceProvider);
        dataSource.setP6spy(properties.getP6spy());
        dataSource.setSeata(properties.getSeata());
        return dataSource;
    }
}
```

## Aop拦截
Aop拦截对应的注解
```shell script
@Aspect
@Component
@Order(0)
public class DynamicDataSourceAspect {
    @Pointcut("@annotation(DatabaseSharding)")
    public void aopPoint() {
    }

    @Before("aopPoint()")
    public void doRouter(ProceedingJoinPoint joinPoint) {
       DatabaseSharding databaseSharding = joinPoint.getTarget().getClass().getAnnotation(DatabaseSharding.class);
       String[] paramNames = ((MethodSignature) joinPoint.getSignature()).getParameterNames();
       Object[] paramValues = joinPoint.getArgs();
       String shardingColumn = "";

       for (int i = 0; i < paramNames.length; i++) {
           if (databaseSharding.value().equals(paramNames[i])) {
               shardingColumn = (String) paramValues[i];
               break;
           }
       }
       DynamicDataSourceHolder.set(shardingColumn);
       try {
           return joinPoint.proceed();
       } catch (Throwable throwable) {
           throwable.printStackTrace();
       }finally {
           DynamicDataSourceHolder.remove();
       }
       return null;
    }
}
```
Mybaits拦截器,替换表名(小提示:可以用一个独立只读权限的数据库账号)
```java
@Intercepts({@Signature(type = StatementHandler.class, method = "prepare", args = {Connection.class, Integer.class})})
public class MybatisInterceptor implements Interceptor {

    private Pattern pattern = Pattern.compile("(from|into|update)[\\s]{1,}(\\w{1,})", Pattern.CASE_INSENSITIVE);

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 获取StatementHandler
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        MetaObject metaObject = MetaObject.forObject(statementHandler, SystemMetaObject.DEFAULT_OBJECT_FACTORY, SystemMetaObject.DEFAULT_OBJECT_WRAPPER_FACTORY, new DefaultReflectorFactory());
        MappedStatement mappedStatement = (MappedStatement) metaObject.getValue("delegate.mappedStatement");

        // 获取自定义注解判断是否进行分表操作
        String id = mappedStatement.getId();
        String className = id.substring(0, id.lastIndexOf("."));
        Class<?> clazz = Class.forName(className);
        DS dbRouterStrategy = clazz.getAnnotation(DS.class);
        if (null != dbRouterStrategy){
            return invocation.proceed();
        }
        
        BoundSql boundSql = statementHandler.getBoundSql();
        String sql = boundSql.getSql();

  
        Matcher matcher = pattern.matcher(sql);
        String tableName = null;
        if (matcher.find()) {
            tableName = matcher.group().trim();
        }
        assert null != tableName;
        String replaceSql = matcher.replaceAll(DynamicDataSourceHolder.get());

        // 通过反射修改SQL语句
        Field field = boundSql.getClass().getDeclaredField("sql");
        field.setAccessible(true);
        field.set(boundSql, replaceSql);
        field.setAccessible(false);
        return invocation.proceed();
    }
}
```

## SpringBoot-Starter

第一步增加META-INF/spring.factories
```shell script
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.xxxx.DataSourceAutoConfig
```
第二步
将你需要加载的所有class都new为Bean,交给Spring管理、可以使用@ConditionalOnXXX条件注解,增加严谨性。
```shell script
@Configuration
public class DataSourceAutoConfig  {

    @Bean(name = "db-router-point")
    @ConditionalOnMissingBean
    public DBRouterJoinPoint point(DBRouterConfig dbRouterConfig, IDBRouterStrategy dbRouterStrategy) {
        return new DBRouterJoinPoint(dbRouterConfig, dbRouterStrategy);
    }

    @Bean
    public DBRouterConfig dbRouterConfig() {
        return new DBRouterConfig(dbCount, tbCount, routerKey);
    }

    @Bean
    public Interceptor plugin() {
        return new DynamicMybaitsPlugin();
    }

```







  