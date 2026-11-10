AOP（Aspect-Oriented Programming，面向切面编程）解决的是这么一类问题：日志、事务、鉴权这些逻辑跟业务无关，却到处都要。AOP 把它们抽成"切面"，在不改动业务代码的前提下织入进去。

Spring AOP 的底层就是**动态代理**。这篇回答两个问题：**代理是在哪一步生成的？JDK 和 CGLIB 怎么选？**

> 代理不是"创建 Bean 的方式"，它是在 Bean 初始化之后额外包一层。Bean 生命周期见 [Spring生命周期.md](Spring生命周期.md)。

# 总览

Spring 创建代理有两个时机：

1. **自定义 `TargetSource`**：在 Bean 实例化**之前**就提前创建代理（少见）；
2. **Bean 初始化完成后**：把原来的 Bean 包装成代理返回（常见，普通 AOP 都走这里）。

核心执行者是 `AbstractAutoProxyCreator`，它实现了 `SmartInstantiationAwareBeanPostProcessor`（进而继承 `BeanPostProcessor`），所以能插手 Bean 的初始化。

# 代理是在哪一步生成的

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
        // （源码注释：Create proxy here if we have a custom TargetSource.
        //  Suppresses unnecessary default instantiation of the target bean.）
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

    protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
        // 收集匹配这个 Bean 的 Advisor（@Aspect → Advisor 的解析就在这里完成）
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
        if (specificInterceptors != DO_NOT_PROXY) {
            this.advisedBeans.put(cacheKey, Boolean.TRUE);
            // 生成 JDK 或 CGLIB 代理
            Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            this.proxyTypes.put(cacheKey, proxy.getClass());
            return proxy;
        }
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;   // 不需要增强，返回原 Bean
    }
}
```

把两个时机理清：

- **`postProcessBeforeInstantiation`**：只在存在**自定义 `TargetSource`** 时提前创建代理，**不负责**解析 `@Aspect`；
- **`postProcessAfterInitialization`**：普通 Bean 的代理在这里生成——`wrapIfNecessary` 先收集匹配的 Advisor（`@Aspect` 到 Advisor 的解析由 `AnnotationAwareAspectJAutoProxyCreator` 完成），再决定是否创建代理。

# AopProxy：JDK 还是 CGLIB

代理统一由 `AopProxy` 抽象出来：

```java
public interface AopProxy {
    Object getProxy();
    Object getProxy(@Nullable ClassLoader classLoader);
}
```

实现有两种，怎么选很关键。

## JDK 动态代理

**面向接口**：如果目标实现了接口，Spring 默认用 JDK 代理。

```java
public class UserServiceProxy implements InvocationHandler {
    private final Object target;

    public UserServiceProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before invoking method: " + method.getName());
        Object result = method.invoke(target, args);      // 反射调用目标方法
        System.out.println("After invoking method: " + method.getName());
        return result;
    }
}

// 生成代理
UserService userService = new UserServiceImpl();
UserService proxy = (UserService) Proxy.newProxyInstance(
        userService.getClass().getClassLoader(),
        userService.getClass().getInterfaces(),           // 必须基于接口
        new UserServiceProxy(userService));
```

实现原理（`Proxy.newProxyInstance` 内部）：

1. 实现 `InvocationHandler` 定义调用处理器；
2. 给 `Proxy` 指定 `ClassLoader` 和一组接口，创建一个动态代理类；
3. 反射拿到这个代理类的构造函数（参数是 `InvocationHandler`）；
4. 用该构造函数创建代理实例，调用处理器作为参数传入。

因为面向接口，**目标没有接口时 JDK 代理无能为力**。

## CGLIB 动态代理

**面向继承**：目标没有接口时用 CGLIB，通过继承目标类、重写方法来增强。

```java
public class ProxyClass implements MethodInterceptor {
    private Object target;

    public Object getProxy(Object target) {
        this.target = target;
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(this.target.getClass());   // 继承目标类
        enhancer.setCallback(this);
        return enhancer.create();
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("Before method invocation");
        Object result = proxy.invokeSuper(obj, args);    // 调用父类（目标）方法
        System.out.println("After method invocation");
        return result;
    }
}
```

CGLIB 基于 ASM 字节码库生成子类，底层靠继承——所以**被 `final` 修饰的类代理不了**。

## 两者对比

| | JDK 动态代理 | CGLIB |
|---|---|---|
| 前提 | 目标实现接口 | 目标有可继承的类（非 final） |
| 原理 | 生成接口的匿名实现类 | 生成目标类的子类 |
| 创建速度 | 快 | 慢 |
| 运行速度 | 较慢 | 较快 |
| 依赖 | JDK 自带 | 需 cglib/asm |

> Spring 的选择：目标有接口默认用 JDK 代理，否则用 CGLIB；也可通过 `proxyTargetClass=true` 强制 CGLIB。

# 小结

1. **Spring AOP 靠什么实现？** 动态代理（JDK / CGLIB）。
2. **代理在哪里生成？** 普通 Bean 在 `postProcessAfterInitialization`（`wrapIfNecessary`）；只有自定义 `TargetSource` 才会在 `postProcessBeforeInstantiation` 提前生成。
3. **`postProcessBeforeInstantiation` 负责解析 `@Aspect` 吗？** 不负责，`@Aspect → Advisor` 的解析在收集候选 Advisor 时完成。
4. **JDK 和 CGLIB 怎么选？** 有接口用 JDK，无接口用 CGLIB；CGLIB 不能代理 `final` 类。
5. **两者性能差异？** JDK 创建快、运行慢；CGLIB 创建慢、运行快。

# 参考

- Spring 5.x 源码：`AbstractAutoProxyCreator`、`AopProxy`、`DefaultAopProxyFactory`
- [Spring生命周期.md](Spring生命周期.md)、[Spring依赖注入.md](Spring依赖注入.md)
