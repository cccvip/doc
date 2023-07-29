spring的另外一个核心是Aop,面向切面编程。面向切面的核心在于动态代理,spring处理的通常是代理方法,而不是代理类。
代理类有两种实现方法,jdk和cglib实现。

```java
用于配置的AOP代理的委托接口，允许创建实际的代理对象。开箱即用的实现适用于JDK动态代理和CGLIB代理
Delegate interface for a configured AOP proxy, allowing for the creation of actual proxy objects.
        Out-of-the-box implementations are available for JDK dynamic proxies and for CGLIB proxies, as applied
public interface AopProxy {
	/**
	 * Create a new proxy object.
	 * <p>Uses the AopProxy's default class loader (if necessary for proxy creation):
	 * usually, the thread context class loader.
	 * @return the new proxy object (never {@code null})
	 * @see Thread#getContextClassLoader()
	 */
	Object getProxy();

	/**
	 * Create a new proxy object.
	 * <p>Uses the given class loader (if necessary for proxy creation).
	 * {@code null} will simply be passed down and thus lead to the low-level
	 * proxy facility's default, which is usually different from the default chosen
	 * by the AopProxy implementation's {@link #getProxy()} method.
	 * @param classLoader the class loader to create the proxy with
	 * (or {@code null} for the low-level proxy facility's default)
	 * @return the new proxy object (never {@code null})
	 */
	Object getProxy(@Nullable ClassLoader classLoader);

}
```
## JDK动态代理：

具体实现原理：

1、通过实现InvocationHandlet接口创建自己的调用处理器

2、通过为Proxy类指定ClassLoader对象和一组interface来创建动态代理

3、通过反射机制获取动态代理类的构造函数，其唯一参数类型就是调用处理器接口类型

4、通过构造函数创建动态代理类实例，构造时调用处理器对象作为参数参入

JDK动态代理是面向接口的代理模式，如果被代理目标没有接口那么Spring也无能为力，

Spring通过java的反射机制生产被代理接口的新的匿名实现类，重写了其中AOP的增强方法。

## CGLib动态代理

CGLib是一个强大、高性能的Code生产类库，可以实现运行期动态扩展java类，Spring在运行期间通过 CGlib继承要被动态代理的类，重写父类的方法，实现AOP面向切面编程呢。

## 两者对比：

- JDK动态代理是面向接口，在创建代理实现类时比CGLib要快，创建代理速度快。

- CGLib动态代理是通过字节码底层继承要代理类来实现（如果被代理类被final关键字所修饰，那么抱歉会失败），在创建代理这一块没有JDK动态代理快，但是运行速度比JDK动态代理要快。




