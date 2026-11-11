Spring IOC 这个词几乎每个面试都会问，但很多人只能说出"控制反转、依赖注入"两个词，再往下问"BeanDefinition 是什么、容器什么时候创建 Bean、FactoryBean 和 BeanFactory 有什么区别"，就说不清了。

这篇按"配置 → BeanDefinition → 容器 → Bean 实例"的顺序，把 IOC 的主链走一遍。Bean 创建的逐行源码在 [Spring生命周期.md](Spring生命周期.md)，这里重点讲 IOC 的整体设计和几个容易混淆的概念。如果只为面试，可以直接滑到最后的**小结**。

> 代理不属于 IOC 的范畴——JDK/CGLIB 动态代理是 AOP 的机制，集中在 [SpringAop代理.md](SpringAop代理.md)，本文不再展开。

# 总览

**一句话：IOC 把"对象创建 + 依赖装配"从业务代码搬到容器；容器先读 BeanDefinition，再按定义创建 Bean。**

```text
配置类 / 注解 / XML
  → BeanDefinitionReader 读取
  → BeanDefinition（类型、作用域、依赖、属性）
  → BeanFactory 注册（beanDefinitionMap）
  → refresh()
      → BeanFactoryPostProcessor 修改 BeanDefinition
      → BeanPostProcessor 注册
      → preInstantiateSingletons
          → getBean → doGetBean → createBean → doCreateBean
  → 单例池 singletonObjects
```

# 核心概念

## IOC 和 DI 是什么

- **IOC（Inversion of Control，控制反转）**：对象的创建和依赖维护不再由业务代码负责，交给容器；
- **DI（Dependency Injection，依赖注入）**：IOC 的具体实现方式——容器把依赖对象注入进来。

业务代码只声明"我需要什么"，容器负责创建、装配、管理生命周期。

## 为什么需要 IOC

假设 MVC 架构，controller 调 service，service 调 dao。最直接的写法是到处 `new`：

```java
public class UserController {
    public void login() {
        UserService userService = new UserService();   // 每次用都 new
        userService.login();
    }
}

public class UserService {
    public void login() {
        UserDao userDao = new UserDao();
        userDao.login();
    }
}
```

问题有两个：

1. **对象创建分散**：每次都 `new`，对象数量暴涨，GC 压力大；用单例能缓解，但依赖关系还是写死的；
2. **横切逻辑无法统一处理**：统计耗时、日志、事务这些逻辑，每层都要硬编码，冗余且难维护。

IOC 把"创建"和"装配"提到容器层，AOP 再把"增强"提到容器层，业务代码只留业务。

## BeanDefinition

Spring 不直接 `new` 对象，而是先把每个 Bean 解析成一份 `BeanDefinition`：类型、作用域、依赖、属性值、初始化/销毁方法都记在里面，容器再照着它创建 Bean。

```java
/**
 * BeanDefinition 描述了一个 bean 实例的属性值、构造参数等信息。
 * 它是一个最小接口，主要目的是允许 BeanFactoryPostProcessor 查看和修改这些元数据。
 */
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
    String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
    String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;

    String getScope();                                   // 作用域
    void setDependsOn(@Nullable String... dependsOn);    // 依赖关系
    ConstructorArgumentValues getConstructorArgumentValues();
    boolean isAutowireCandidate();                       // 是否参与自动装配
}
```

## BeanFactory / ApplicationContext / FactoryBean

| 概念 | 是什么 |
|---|---|
| `BeanFactory` | 容器本体，负责 `getBean`、管理 BeanDefinition |
| `ApplicationContext` | `BeanFactory` 的超集，多了事件、国际化、资源加载等能力 |
| `FactoryBean` | 一个特殊的 Bean，`getObject()` 返回真正的对象 |
| `BeanDefinition` | Bean 的定义（类型、作用域、依赖、属性） |

这三个最容易混的是 `BeanFactory` 和 `FactoryBean`：

- `BeanFactory` 是**容器**；
- `FactoryBean` 是**容器里的一个 Bean**，用来生产另一个对象。

```java
public interface FactoryBean<T> {
    T getObject() throws Exception;   // 返回真正的对象
    Class<?> getObjectType();
    boolean isSingleton();
}
```

取 Bean 时有个细节：`getBean("xxx")` 返回 `getObject()` 的结果，`getBean("&xxx")` 才返回 `FactoryBean` 本身。

## BeanDefinitionReader

负责把配置读成 BeanDefinition：

- **`AnnotatedBeanDefinitionReader`**：注解版入口，SpringBoot 扫描 `@Component` 等；
- **`XmlBeanDefinitionReader`**：解析 XML 里的 `<bean>`。

# 源码分析

## 从配置到 BeanDefinition：registerBeanDefinition

所有的 BeanDefinition 最终都注册进 `DefaultListableBeanFactory`：

```java
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) {
    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");

    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    if (existingDefinition != null) {
        // 同名 Bean：默认禁止覆盖，抛 BeanDefinitionOverrideException
        if (!isAllowBeanDefinitionOverriding()) {
            throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
        }
        this.beanDefinitionMap.put(beanName, beanDefinition);
    } else {
        // 新 Bean：放进 map，同时维护一个有序的 names 列表
        this.beanDefinitionMap.put(beanName, beanDefinition);
        this.beanDefinitionNames.add(beanName);
        removeManualSingletonName(beanName);
    }

    // 定义变了，清掉按类型查找的缓存
    if (existingDefinition != null || containsSingleton(beanName)) {
        resetBeanDefinition(beanName);
    }
}
```

两个核心字段：

- `beanDefinitionMap`：`beanName → BeanDefinition`，注册表本体；
- `beanDefinitionNames`：保持注册顺序，`preInstantiateSingletons` 按这个顺序创建。

## refresh() 的骨架

`AbstractApplicationContext.refresh()` 是容器启动的总入口，和 Bean 相关的主要是这几步：

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        prepareRefresh();
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        prepareBeanFactory(beanFactory);

        try {
            postProcessBeanFactory(beanFactory);

            // ① 执行 BeanFactoryPostProcessor：修改 BeanDefinition
            invokeBeanFactoryPostProcessors(beanFactory);

            // ② 注册 BeanPostProcessor：初始化前后的扩展点
            registerBeanPostProcessors(beanFactory);

            initMessageSource();
            initApplicationEventMulticaster();
            onRefresh();
            registerListeners();

            // ③ 创建所有非懒加载的单例
            finishBeanFactoryInitialization(beanFactory);

            finishRefresh();
        } catch (BeansException ex) {
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        } finally {
            resetCommonCaches();
        }
    }
}
```

顺序不能乱：**先改 BeanDefinition，再注册 BeanPostProcessor，最后才创建 Bean**。前两步放到创建之后，就赶不上这次创建了。

## doGetBean：从缓存到创建

`finishBeanFactoryInitialization` 最终落到 `preInstantiateSingletons`，它遍历 `beanDefinitionNames` 并调用 `getBean`。`getBean` 的核心在 `doGetBean`：

```java
protected <T> T doGetBean(String name, @Nullable Class<T> requiredType,
        @Nullable Object[] args, boolean typeCheckOnly) throws BeansException {

    String beanName = transformedBeanName(name);
    Object bean;

    // ① 先查单例缓存（三级缓存、循环依赖都在这里，详见 Spring循环依赖.md）
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        // ② prototype 循环依赖检测
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // ③ 父容器有，就委托父容器
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            return (T) parentBeanFactory.getBean(nameToLookup, requiredType);
        }

        // ④ 合并 BeanDefinition（处理父子定义、占位符等）
        RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

        // ⑤ 先初始化 depends-on 的 Bean
        String[] dependsOn = mbd.getDependsOn();
        if (dependsOn != null) {
            for (String dep : dependsOn) {
                registerDependentBean(dep, beanName);
                getBean(dep);
            }
        }

        // ⑥ 按作用域创建
        if (mbd.isSingleton()) {
            sharedInstance = getSingleton(beanName, () -> {
                try {
                    return createBean(beanName, mbd, args);
                } catch (BeansException ex) {
                    destroySingleton(beanName);
                    throw ex;
                }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        } else if (mbd.isPrototype()) {
            Object prototypeInstance = createBean(beanName, mbd, args);
            bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
        } else {
            // request / session 等作用域
            // ...
        }
    }
    return (T) bean;
}
```

## createBean / doCreateBean：概览

`createBean` 先给 `BeanPostProcessor` 一次"提前返回代理"的机会（`resolveBeforeInstantiation`），然后进入 `doCreateBean`：

- 实例化 → 收集注入点 → 提前暴露 → 属性填充 → 初始化 → 注册销毁。

逐行源码和每个扩展点的时机，见 [Spring生命周期.md](Spring生命周期.md)，这里不重复。

## FactoryBean：getBean 的特殊分支

`getBean` 最后会调用 `getObjectForBeanInstance`。如果拿到的 `sharedInstance` 是 `FactoryBean`，并且 beanName 没有 `&` 前缀，就会调用它的 `getObject()`：

```java
// 简化示意
if (beanInstance instanceof FactoryBean) {
    if (!BeanFactoryUtils.isFactoryDereference(name)) {
        // getBean("xxx") → 返回 getObject() 的结果
        return ((FactoryBean<?>) beanInstance).getObject();
    }
}
// getBean("&xxx") → 返回 FactoryBean 本身
return beanInstance;
```

这就是 `FactoryBean` 和普通 Bean 的区别：**容器里注册的是 `FactoryBean`，但 `getBean` 默认返回它生产出来的对象。**

# 实战与常见坑

## 场景：用 FactoryBean 注册 Mapper 接口

MyBatis 的 Mapper 没有实现类，却能被注入，靠的就是 `FactoryBean` + 动态代理：

```java
public class MapperFactoryBean<T> implements FactoryBean<T> {
    private final Class<T> mapperInterface;

    public MapperFactoryBean(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    @Override
    @SuppressWarnings("unchecked")
    public T getObject() {
        // 为接口生成 JDK 动态代理，方法调用转成 SQL 执行
        return (T) Proxy.newProxyInstance(
                mapperInterface.getClassLoader(),
                new Class[]{mapperInterface},
                (proxy, method, args) -> {
                    // 这里省略：解析 method 对应的 SQL 并执行
                    return null;
                });
    }

    @Override
    public Class<T> getObjectType() {
        return mapperInterface;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

把它注册成 Bean 后，业务代码 `@Autowired UserMapper userMapper` 拿到的就是 `getObject()` 返回的代理对象——这也是为什么 Mapper 不需要实现类。

## 常见坑

1. **包扫描不到**：`@ComponentScan` 范围、启动类所在包、`@Component` 缺失，都会导致 BeanDefinition 根本没注册。
2. **同名 Bean 覆盖**：Spring Boot 2.1+ 默认禁止覆盖，会抛 `BeanDefinitionOverrideException`；要覆盖需显式开启 `spring.main.allow-bean-definition-overriding=true`。
3. **取 `FactoryBean` 本身忘了加 `&`**：`getBean("xxx")` 返回 `getObject()`，`getBean("&xxx")` 才返回 `FactoryBean`。
4. **`BeanPostProcessor` 里注入另一个 `BeanPostProcessor`**：容易触发过早初始化，必要时用 `static @Bean` 方法或 `ObjectProvider` 延迟获取。
5. **`@Configuration(proxyBeanMethods = true)` 的代理**：默认会用 CGLIB 代理配置类，保证 `@Bean` 方法返回单例；设为 `false` 后，直接调用 `@Bean` 方法会创建新对象。
6. **prototype 注入到 singleton**：只会注入一次，之后拿到的都是同一个对象；想每次拿新的，用 `ObjectProvider` 或 `@Lookup`。
7. **循环依赖**：三级缓存的细节见 [Spring循环依赖.md](Spring循环依赖.md)。

# 小结

我一向不太喜欢写总结，因为该说的正文里都说了。不过考虑到很多读者是冲着面试来的，还是列成清单：

1. **IOC 解决什么？** 把对象创建与依赖维护从业务代码挪进容器，降低耦合、方便替换与测试。
2. **为什么要 BeanDefinition？** 容器先把 Bean 解析成定义，再照定义创建；这样 `BeanFactoryPostProcessor` 才有机会修改它。
3. **容器什么时候创建 Bean？** `refresh()` 的 `finishBeanFactoryInitialization` 阶段，遍历非懒加载单例，走 `getBean → doGetBean → createBean`。
4. **BeanFactory 和 FactoryBean 的区别？** 前者是容器，后者是容器里生产对象的特殊 Bean。
5. **`getBean("xxx")` 和 `getBean("&xxx")` 的区别？** 前者返回 `getObject()`，后者返回 `FactoryBean` 本身。
6. **几个扩展点的时机？** `BeanFactoryPostProcessor`（实例化前，改定义）、`BeanPostProcessor`（初始化前后，改实例）、`InitializingBean`（初始化）、`DisposableBean`（销毁）。
7. **为什么 refresh 的顺序不能乱？** 必须先改 BeanDefinition、再注册 BeanPostProcessor，最后才创建 Bean。

# 参考

- [Spring Framework 官方文档：The IoC Container](https://docs.spring.io/spring-framework/reference/core/beans.html)
- Spring 5.x 源码：`DefaultListableBeanFactory`、`AbstractApplicationContext.refresh`、`FactoryBean`
- [基本概念.md](基本概念.md)、[Spring生命周期.md](Spring生命周期.md)、[SpringAop代理.md](SpringAop代理.md)
