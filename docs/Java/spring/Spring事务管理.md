所有源码基于SpringBootV2.5.7版本,

Spring的事务管理,使用@EnableTransactionManagement,源码流程分析如下
1. 开启事务(@EnableTransactionManagement),使用@Import注解引入AutoProxyRegistrar,ProxyTransactionManagementConfiguration.
2. AutoProxyRegistrar生成需要Proxy的Bean,并将其注册到Bean工厂中
3. ProxyTransactionManagementConfiguration事务处理配置类,其中配置了Aop执行的切入点(配置@Transactional的方法)
```java
abstract class TransactionAttributeSourcePointcut extends StaticMethodMatcherPointcut implements Serializable {
	@Override
	public boolean matches(Method method, Class<?> targetClass) {
		TransactionAttributeSource tas = getTransactionAttributeSource();
		return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
	}
}
```
4. TransactionInterceptor代理增强执行函数
```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {
	@Override
	@Nullable
	public Object invoke(MethodInvocation invocation) throws Throwable {
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
		//代理类执行方法
		return invokeWithinTransaction(invocation.getMethod(), targetClass, new CoroutinesInvocationCallback() {
			@Override
			@Nullable
			public Object proceedWithInvocation() throws Throwable {
				return invocation.proceed();
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
核心逻辑在invokeWithinTransaction中执行,代码逻辑比较长,可以简化一下。
1. 创建事务,并且ThreadLocal存放TransactionInfo
```shell script
protected TransactionInfo prepareTransactionInfo(@Nullable PlatformTransactionManager tm,
			@Nullable TransactionAttribute txAttr, String joinpointIdentification,
			@Nullable TransactionStatus status) {
    TransactionInfo txInfo = new TransactionInfo(tm, txAttr, joinpointIdentification);
        //....省略
        txInfo.bindToThread();
		return txInfo;
	  }
	  private void bindToThread() {
			//如果上下文存在一个旧TransactionInfo
			this.oldTransactionInfo = transactionInfoHolder.get();
			 //则使用新的事务
            transactionInfoHolder.set(this);
		}
```
2. 执行doBegin(),设置connection.setAutoCommit(false)

3. 执行@Tranactional包含的方法

3. 事务提交/或者事务回滚

4. 清理TransactionInfo
```shell
	private void restoreThreadLocalStatus() {
			//将上一个事务的信息存放在当前
			transactionInfoHolder.set(this.oldTransactionInfo);
		}
```

@Transactional作用域范围,必须是public,否则就会跳过
```java
public abstract class AbstractFallbackTransactionAttributeSource
		implements TransactionAttributeSource, EmbeddedValueResolverAware {
	protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
		// Don't allow no-public methods as required.
		if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
			return null;
		}
        //省略
    }
}







