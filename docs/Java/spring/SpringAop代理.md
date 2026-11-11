`@Transactional`、`@Async`、自定义日志切面，背后都是同一个东西：Spring AOP 生成的代理对象。很多人能说出"JDK 代理基于接口、CGLIB 基于继承"，但再问一句"代理是在哪一步替换掉原 Bean 的、方法调用怎么进到拦截器链"，就卡住了。

这篇从 `AbstractAutoProxyCreator` 和 `JdkDynamicAopProxy` 的源码出发，把"代理怎么生成、调用怎么被拦截"走一遍。如果只为面试，可以直接滑到最后的**小结**。

# 总览

**一句话：Spring AOP 在 Bean 初始化之后，用 JDK 动态代理或 CGLIB 生成代理对象替换原 Bean；调用代理方法时，先走拦截器链，最后才反射调用目标方法。**

```text
启动阶段：
AbstractAutoProxyCreator.postProcessAfterInitialization
  → wrapIfNecessary
      → getAdvicesAndAdvisorsForBean        收集匹配的 Advisor
      → createProxy
          → DefaultAopProxyFactory.createAopProxy
              ├─ 有接口且未开 proxyTargetClass → JdkDynamicAopProxy
              └─ 无接口或 proxyTargetClass=true → CglibAopProxy
  → 代理对象替换原 Bean，放入容器

调用阶段：
proxy.method()
  → JdkDynamicAopProxy.invoke / CglibAopProxy.intercept
      → getInterceptorsAndDynamicInterceptionAdvice   拿到拦截器链
      → ReflectiveMethodInvocation.proceed            责任链
          → 拦截器 before → 目标方法 → 拦截器 after
```

> 代理不是"创建 Bean 的方式"，它是在 Bean 初始化之后额外包一层。Bean 生命周期见 [Spring生命周期.md](Spring生命周期.md)。

# 核心概念

## AOP 里的几个角色

| 术语 | 含义 |
|---|---|
| Aspect（切面） | 横切逻辑的模块，比如日志切面 |
| Pointcut（切点） | 匹配哪些方法 |
| Advice（通知） | 在方法前 / 后 / 环绕做什么 |
| Advisor | Pointcut + Advice 的组合，Spring AOP 内部真正持有的东西 |
| Target（目标对象） | 被代理的原始 Bean |
| Proxy（代理对象） | 增强后的对象，放进容器 |

`@Aspect` 本身不是 Advisor，它需要被 `AnnotationAwareAspectJAutoProxyCreator` 解析成一个个 Advisor，才能参与匹配。

## 代理生成的两个时机

1. **自定义 `TargetSource`**：在 Bean 实例化**之前**就提前创建代理（少见）；
2. **Bean 初始化完成后**：把原来的 Bean 包装成代理返回（常见，普通 AOP 都走这里）。

核心执行者是 `AbstractAutoProxyCreator`，它实现了 `SmartInstantiationAwareBeanPostProcessor`（进而继承 `BeanPostProcessor`），所以能插手 Bean 的初始化。

## JDK 还是 CGLIB

| 条件 | 代理方式 |
|---|---|
| 目标实现了接口，且未开 `proxyTargetClass` | JDK 动态代理 |
| 目标没有接口 | CGLIB |
| `proxyTargetClass=true` | CGLIB（即使有接口） |
| 目标是接口 / 已经是 JDK 代理 | JDK |

> 容易记混的一点：原生 Spring 默认"有接口就用 JDK"；但 **Spring Boot 2.x 默认 `spring.aop.proxy-target-class=true`，也就是默认 CGLIB**，即使 Bean 实现了接口。这是很多人"明明有接口却用了 CGLIB"的原因。

# 源码分析

## 入口：postProcessAfterInitialization

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
        implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {

    // 时机①：实例化前——只在存在自定义 TargetSource 时提前创建代理
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

        // 只有给了自定义 TargetSource，才在这里提前创建代理
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

        return null;   // 普通 Bean 返回 null，走正常流程
    }

    // 时机②：初始化后——普通 Bean 的代理在这里生成
    @Override
    public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
        if (bean != null) {
            Object cacheKey = getCacheKey(bean.getClass(), beanName);
            // 如果这个 Bean 已经在循环依赖里提前生成过代理，就不再重复包装
            if (this.earlyProxyReferences.remove(cacheKey) != bean) {
                return wrapIfNecessary(bean, beanName, cacheKey);
            }
        }
        return bean;
    }
}
```

两个时机的区别：

- **`postProcessBeforeInstantiation`**：只在存在**自定义 `TargetSource`** 时提前创建代理，**不负责**解析 `@Aspect`；
- **`postProcessAfterInitialization`**：普通 Bean 的代理在这里生成——`wrapIfNecessary` 先收集匹配的 Advisor（`@Aspect` 到 Advisor 的解析由 `AnnotationAwareAspectJAutoProxyCreator` 完成），再决定是否创建代理。

## wrapIfNecessary：收集 Advisor、决定是否代理

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    // 这些 Bean 不需要代理：已经处理过的、基础设施类、切面本身
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // 找出匹配这个 Bean 的所有 Advisor（@Aspect → Advisor 的解析就在这里完成）
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);

    // 没有匹配的切面，直接返回原 Bean
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // 有切面：生成 JDK 或 CGLIB 代理，替换原 Bean
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

## createProxy：交给 ProxyFactory

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
        @Nullable Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }

    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);

    if (proxyFactory.isProxyTargetClass()) {
        // 开了 proxyTargetClass：如果是 JDK 代理类，仍然要把接口加进去
        if (Proxy.isProxyClass(beanClass)) {
            for (Class<?> ifc : beanClass.getInterfaces()) {
                proxyFactory.addInterface(ifc);
            }
        }
    } else {
        // 没开 proxyTargetClass：看是否需要强制 CGLIB，否则收集目标实现的接口
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        } else {
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    // 最终由 ProxyFactory 决定用 JDK 还是 CGLIB
    return proxyFactory.getProxy(getProxyClassLoader());
}
```

## createAopProxy：JDK 还是 CGLIB

`ProxyFactory.getProxy` 最终会进 `DefaultAopProxyFactory.createAopProxy`：

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    // 没开 optimize / proxyTargetClass，并且用户提供了接口 → JDK
    if (!config.isOptimize() && !config.isProxyTargetClass() && !hasNoUserSuppliedProxyInterfaces(config)) {
        return new JdkDynamicAopProxy(config);
    }

    Class<?> targetClass = config.getTargetClass();
    if (targetClass == null) {
        throw new AopConfigException("TargetSource cannot determine target class: ...");
    }

    // 目标是接口 / 已经是 JDK 代理 → 只能 JDK
    if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
        return new JdkDynamicAopProxy(config);
    }

    // 其余走 CGLIB
    return new ObjenesisCglibAopProxy(config);
}
```

这段就是"怎么选"的全部答案：**先看有没有接口、有没有强制 `proxyTargetClass`，再看目标本身是不是接口，最后落到 CGLIB。**

## JdkDynamicAopProxy.invoke：拦截器链

JDK 代理的调用入口是 `InvocationHandler.invoke`：

```java
@Override
@Nullable
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;

    TargetSource targetSource = this.advised.targetSource;
    Object target = null;

    try {
        // equals / hashCode 等 Object 方法直接处理，不走拦截器
        if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
            return equals(args[0]);
        }
        if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
            return hashCode();
        }
        if (method.getDeclaringClass() == DecoratingProxy.class) {
            return AopProxyUtils.ultimateTargetClass(this.advised);
        }

        Object retVal;
        if (this.advised.exposeProxy) {
            // 把代理暴露到 ThreadLocal，供 AopContext.currentProxy() 使用
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);

        // ① 拿到这个方法匹配的拦截器链
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

        if (chain.isEmpty()) {
            // ② 没有拦截器：直接反射调用目标方法
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
        } else {
            // ③ 有拦截器：构造 ReflectiveMethodInvocation，责任链执行
            MethodInvocation invocation =
                    new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
            retVal = invocation.proceed();
        }

        return retVal;
    } finally {
        if (target != null && !targetSource.isStatic()) {
            targetSource.releaseTarget(target);
        }
        if (setProxyContext) {
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```

## ReflectiveMethodInvocation.proceed：责任链

`proceed()` 是 Spring AOP 拦截器链的核心，本质是一个递归的责任链：

```java
@Override
@Nullable
public Object proceed() throws Throwable {
    // 拦截器都执行完了，最后调用目标方法
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();
    }

    Object interceptorOrInterceptionAdvice =
            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);

    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
        InterceptorAndDynamicMethodMatcher dm = (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
        Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
        if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
            return dm.interceptor.invoke(this);
        } else {
            // 运行时不匹配，跳过这个拦截器
            return proceed();
        }
    } else {
        return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    }
}

protected Object invokeJoinpoint() throws Throwable {
    return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
}
```

每个拦截器（比如 `TransactionInterceptor`）在 `invoke` 里做自己的事，然后调 `invocation.proceed()` 把链往下走；全部走完后才反射调用目标方法。

## CglibAopProxy：核心也是同一套

CGLIB 代理的入口是 `DynamicAdvisedInterceptor.intercept`，逻辑和 JDK 版本几乎一样：拿拦截器链，有链就构造 `CglibMethodInvocation` 执行，没链就直接调用目标方法。

```java
@Override
@Nullable
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;
    Object target = null;
    TargetSource targetSource = this.advised.getTargetSource();

    try {
        if (this.advised.exposeProxy) {
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }
        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);

        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

        Object retVal;
        if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = methodProxy.invoke(target, argsToUse);
        } else {
            retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
        }
        return retVal;
    } finally {
        if (target != null && !targetSource.isStatic()) {
            targetSource.releaseTarget(target);
        }
        if (setProxyContext) {
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```

区别只在"怎么调用目标方法"：JDK 用反射 `Method.invoke`，CGLIB 用 `MethodProxy.invoke`；拦截器链和 `proceed()` 是同一套。

# 实战与常见坑

## 场景：同类自调用，事务为什么没生效

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

调用 `create()` 时，`this.pay()` 直接走原始对象，根本不会经过代理，所以 `pay()` 上的事务拦截器不会执行。日志、`@Async`、自定义切面同理。

三种解决方式：

1. **注入自身代理**：`@Autowired private OrderService self;`，然后 `self.pay()`；
2. **用 `AopContext.currentProxy()`**：需要开启 `@EnableAspectJAutoProxy(exposeProxy = true)`；
3. **拆到另一个 Bean**：把 `pay()` 放到独立的服务里，通过注入调用。

> `AopContext.currentProxy()` 依赖 ThreadLocal，异步线程或线程池里拿不到。

## 常见坑

1. **同类自调用不走代理**：事务、日志、`@Async` 都会失效，这是 AOP 代理的通病。
2. **`final` 类 / 方法**：CGLIB 靠继承，`final` 类和方法无法重写，增强失效。
3. **`private` 方法**：JDK 代理只代理接口方法，CGLIB 也拦不到 `private` 方法。
4. **JDK 代理只认接口**：按实现类注入会抛 `BeanNotOfRequiredTypeException`，要么按接口注入，要么开 `proxyTargetClass=true`。
5. **Spring Boot 默认 CGLIB**：`spring.aop.proxy-target-class=true` 是默认值，和"有接口就用 JDK"的旧印象不同。
6. **切面本身不会被代理**：`isInfrastructureClass` / `shouldSkip` 会跳过 `@Aspect` 类。
7. **代理对象不是原始对象**：自己 `new` 出来的对象不受容器管理；只有通过代理调用，事务等增强才会生效。

# 小结

我一向不太喜欢写总结，因为该说的正文里都说了。不过考虑到很多读者是冲着面试来的，还是列成清单：

1. **Spring AOP 靠什么实现？** 动态代理（JDK / CGLIB）。
2. **代理在哪里生成？** 普通 Bean 在 `postProcessAfterInitialization`（`wrapIfNecessary`）；只有自定义 `TargetSource` 才会在 `postProcessBeforeInstantiation` 提前生成。
3. **JDK 和 CGLIB 怎么选？** 有接口且未开 `proxyTargetClass` 用 JDK；无接口或开了 `proxyTargetClass` 用 CGLIB。
4. **Spring Boot 默认用哪个？** 默认 `proxy-target-class=true`，优先 CGLIB。
5. **调用怎么进拦截器链？** `JdkDynamicAopProxy.invoke` 取出拦截器链，交给 `ReflectiveMethodInvocation.proceed` 责任链执行，最后反射调用目标方法。
6. **为什么自调用会失效？** `this` 是原始对象，不是代理，拦截器没有机会介入。
7. **`final` / `private` 方法为什么不行？** CGLIB 无法重写 `final`，JDK 和 CGLIB 都拦不到 `private`。

# 参考

- [Spring Framework 官方文档：Aspect Oriented Programming](https://docs.spring.io/spring-framework/reference/core/aop.html)
- Spring 5.x 源码：`AbstractAutoProxyCreator`、`DefaultAopProxyFactory`、`JdkDynamicAopProxy`、`ReflectiveMethodInvocation`
- [Spring生命周期.md](Spring生命周期.md)、[Spring依赖注入.md](Spring依赖注入.md)
