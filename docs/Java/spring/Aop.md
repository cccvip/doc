# what
Spring AOP（Aspect-Oriented Programming）是Spring框架中的一个重要特性，它是一种面向切面编程的技术。AOP通过在程序运行的不同阶段，将横切关注点（如日志记录、事务管理等）从主业务逻辑中分离出来，以模块化的方式实现对横切关注点的处理。

在Spring AOP中，通过定义切点（Pointcut）来指定在哪些方法和位置插入横切逻辑，然后使用切面（Aspect）定义具体的横切逻辑。Spring AOP提供了多种通知（Advice）类型，包括前置通知（Before）、后置通知（After）、环绕通知（Around）等，用于在切点前、后或周围执行特定的逻辑。

Spring AOP的优势在于它能够在不修改原有业务逻辑代码的情况下，通过切面的方式对系统进行功能增强或横切关注点的处理，提供了更好的代码可维护性和可重用性。


# how
SpringAop创建代理的时机,有两种
1 TargetSource 
2 Bean实例化完成后,返回Aop-Bean,而非原来的Bean

那么Spring是如何实现Aop,我们在IOC的章节了解到Bean的生命周期,知道Spring扩展了很多接口,以便扩展。Aop自然也不例外,核心代码

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
		  
        //省略其它
    	@Override
    	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
    		Object cacheKey = getCacheKey(beanClass, beanName);
    
    		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
    			if (this.advisedBeans.containsKey(cacheKey)) {
    				return null;
    			}
    			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
    				this.advisedBeans.put(cacheKey, Boolean.FALSE);
    				return null;
    			}
    		}
    
    		// Create proxy here if we have a custom TargetSource.
    		// Suppresses unnecessary default instantiation of the target bean:
    		// The TargetSource will handle target instances in a custom fashion.
    		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
    		if (targetSource != null) {
    			if (StringUtils.hasLength(beanName)) {
    				this.targetSourcedBeans.add(beanName);
    			}
    			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
    			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
    			this.proxyTypes.put(cacheKey, proxy.getClass());
    			return proxy;
    		}
    
    		return null;
    	}
	@Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyProxyReferences.remove(cacheKey) != bean) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
    	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    		//省略
    		// Create proxy if we have advice.
    		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    		if (specificInterceptors != DO_NOT_PROXY) {
    			this.advisedBeans.put(cacheKey, Boolean.TRUE);
                //jdk/cglib动态代理
    			Object proxy = createProxy(
    					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
    			
                this.proxyTypes.put(cacheKey, proxy.getClass());
    			return proxy;
    		}
    
    		this.advisedBeans.put(cacheKey, Boolean.FALSE);
    		return bean;
    	}
}
```
AbstractAutoProxyCreator实现了两类接口，BeanFactoryAware和BeanPostProcessor
- postProcessBeforeInstantiation：主要是处理使用了@Aspect注解的切面类，然后将切面类的所有切面方法根据使用的注解生成对应Advice，并将Advice连同切入点匹配器和切面类等信息一并封装到Advisor
- ProcessAfterInitialization：主要负责将Advisor注入到合适的位置，创建代理（cglib或jdk)，为后面给代理进行增强实现做准备。




动态代理主要有两种实现方式(cglib或jdk)
```java
public interface AopProxy {

	Object getProxy();

	Object getProxy(@Nullable ClassLoader classLoader);

}
```

### JDK动态代理：

具体实现原理：

1、通过实现InvocationHandlet接口创建自己的调用处理器

2、通过为Proxy类指定ClassLoader对象和一组interface来创建动态代理

3、通过反射机制获取动态代理类的构造函数，其唯一参数类型就是调用处理器接口类型

4、通过构造函数创建动态代理类实例，构造时调用处理器对象作为参数参入

JDK动态代理是面向接口的代理模式，如果被代理目标没有接口那么Spring也无能为力，

Spring通过java的反射机制生产被代理接口的新的匿名实现类，重写了其中AOP的增强方法。

### CGLib动态代理

CGLib是一个强大、高性能的Code生产类库，可以实现运行期动态扩展java类，Spring在运行期间通过 CGlib继承要被动态代理的类，重写父类的方法，实现AOP面向切面编程呢。

## 两者对比：

- JDK动态代理是面向接口，在创建代理实现类时比CGLib要快，创建代理速度快。

- CGLib动态代理是通过字节码底层继承要代理类来实现（如果被代理类被final关键字所修饰，那么抱歉会失败），在创建代理这一块没有JDK动态代理快，但是运行速度比JDK动态代理要快。




