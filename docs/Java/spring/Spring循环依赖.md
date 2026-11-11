`AService` 依赖 `BService`，`BService` 又依赖 `AService`，这种互相依赖在 Spring 项目里很常见，也是面试高频。很多人能背出"三级缓存"，但说不清每一级在哪一步写入、哪一步读取，以及为什么两级不够。

这篇从 `doGetBean` 和 `getSingleton` 的源码出发，把 A ↔ B 的创建过程走一遍。如果只为面试，可以直接滑到最后的**小结**。

# 总览

**一句话：三级缓存的本质，是让 Bean 在还没初始化完成时就能被提前引用，同时把"要不要生成代理"推迟到真正被引用的那一刻。**

先看能不能解决：

| 场景 | 能否解决 |
|---|---|
| 单例 + 字段 / setter 注入 | 能 |
| 单例 + 构造器注入 | 不能 |
| prototype 作用域 | 不能 |

整体链路：

```text
doGetBean
  └─ getSingleton(beanName)                  查缓存
        ├─ singletonObjects                  一级：完整 Bean
        ├─ earlySingletonObjects             二级：提前暴露的对象 / 代理
        └─ singletonFactories                三级：ObjectFactory，延迟生成
  └─ getSingleton(beanName, factory)          没命中才创建
        ├─ beforeSingletonCreation            标记"正在创建"
        ├─ createBean → doCreateBean
        │     └─ addSingletonFactory          实例化后、填充属性前，写入三级
        └─ addSingleton                       创建完成，放入一级并清理二三级
```

# 核心概念

## 什么是循环依赖

对象之间的依赖关系形成了一个环：A 的字段里需要 B，B 的字段里又需要 A。画出来是 `A → B → A`；也可能是 `A → B → C → A` 这种间接的环。

用代码表示：

```java
class A {
    B b;
}

class B {
    A a;
}
```

## 为什么会出问题

容器创建对象是一个接一个来的：要创建 A，得先把 A 依赖的 B 创建好；要创建 B，又得先把 B 依赖的 A 创建好。于是"创建 A"的过程里又在等 A，形成永远等不到的死循环，最后只能报错或者无限递归。

根源不是"对象之间不能互相依赖"，而是**创建顺序**：如果 A、B 都能先创建出来、再互相设置引用，这个环其实可以解开。Spring 的三级缓存做的就是这件事——让 A 在还没完全创建好时，就能先被 B 拿到。

## 只用一个 Map 够不够

先别急着上三级。写一个最简容器：只有一个 `Map`，对象实例化后先放进去，再填属性：

```java
public class Singleton {
    private static final Map<String, Object> singleMap = new ConcurrentHashMap<>();

    public static void main(String[] args) throws Exception {
        System.out.println(getBean(B.class).getA());
        System.out.println(getBean(A.class).getB());
    }

    private static <T> T getBean(Class<T> beanClass) throws Exception {
        String beanName = beanClass.getSimpleName().toLowerCase();
        if (singleMap.containsKey(beanName)) {
            return (T) singleMap.get(beanName);
        }
        Object obj = beanClass.newInstance();
        singleMap.put(beanName, obj);             // ① 先入缓存：半成品也能被取到
        for (Field field : obj.getClass().getDeclaredFields()) {
            field.setAccessible(true);
            Class<?> fieldClass = field.getType();
            String fieldBeanName = fieldClass.getSimpleName().toLowerCase();
            field.set(obj, singleMap.containsKey(fieldBeanName)
                    ? singleMap.get(fieldBeanName)              // ② 已在缓存里，直接取（打破循环的关键）
                    : getBean(fieldClass));
        }
        return (T) obj;
    }
}
```

跑出来 A 和 B 能互相持有，**没报错**：

```text
com.xiao.A@4f3f5b24
com.xiao.B@15aeb7ab
```

所以"一级缓存解决不了循环依赖"这个说法**并不准确**：对普通字段注入，一个 Map 就够——**实例化后立刻放进 Map，再填属性**，对方来取时就能拿到这个半成品（对象已经 new 出来，但属性还没填完）。

那 Spring 为什么还要三级？因为它还要处理 AOP 代理，而代理不该在这个时间点生成。下面开始看源码。

## 为什么需要三级缓存

![img.png](_assets/beanRecyle.png)

真正的原因是 **AOP 代理**：

- 循环依赖时，Spring 得在 Bean 还没初始化完成时就把引用**提前暴露**给对方；
- 但代理对象本该在**初始化之后**由 `BeanPostProcessor` 生成；
- 只有一个 Map，就无法区分"原始对象"和"最终代理对象"，也没法在正确的时机把代理换进去。

于是才需要分级：一级放完整 Bean、二级放提前暴露的对象、三级放一个"能产出对象的工厂"。

### 二级缓存为什么不够

两个缓存（完整 Bean + 提前暴露的对象）——对普通循环依赖确实够。Spring 没用两级，还是因为 AOP：

如果二级缓存里放的是**原始对象**，那提前暴露出去的就是原始对象；等后续代理生成后，别的 Bean 里注入的引用可能仍指向原始对象，导致增强失效。

所以需要第三级缓存：**放的不是对象，而是一个 `ObjectFactory`（Lambda）**，等到真正被依赖时才执行，从而**延迟决定"要不要提前生成代理"**。

### 三级缓存里放的是什么

三级缓存 `singletonFactories` 里是一个 Lambda：

```java
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```

它的作用就是延迟决策：只有真出现循环依赖、有人来取时，才执行这个 Lambda；执行时会走 `getEarlyBeanReference`——如果该 Bean 需要 AOP，就在这里**提前生成代理**并放入二级缓存，否则直接返回原始对象。

## 为什么构造器和 prototype 解不了

- **构造器循环依赖**：依赖对象必须在构造方法里就初始化好。而三级缓存生效的前提是"实例化完、属性还没填"这个中间态能被提前暴露——构造器注入时，对象连实例都没造出来，自然没得暴露。
- **prototype 循环依赖**：prototype 每次都是 `new` 出来的，不放进缓存。而循环依赖的解法完全依赖缓存，所以解不了。

# 源码分析

## doGetBean：先查缓存

`getBean` 最终会走到 `AbstractBeanFactory.doGetBean`。它先尝试从缓存拿，拿不到才创建：

```java
protected <T> T doGetBean(String name, @Nullable Class<T> requiredType,
        @Nullable Object[] args, boolean typeCheckOnly) throws BeansException {

    String beanName = transformedBeanName(name);
    Object bean;

    // ① 先查缓存：一级 → 二级 → 三级（三级会执行 ObjectFactory）
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // 命中：可能是完整 Bean，也可能是提前暴露的半成品 / 代理
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        // ② 没命中，进入创建流程
        // ...
        if (mbd.isSingleton()) {
            sharedInstance = getSingleton(beanName, () -> {
                try {
                    return createBean(beanName, mbd, args);   // 最终到 doCreateBean
                } catch (BeansException ex) {
                    destroySingleton(beanName);
                    throw ex;
                }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }
        // ...
    }
    return (T) bean;
}
```

关键就是两个重载：`getSingleton(beanName)` 负责**读**，`getSingleton(beanName, factory)` 负责**创建并登记**。

## getSingleton：三级查找

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 一级：完整 Bean
    Object singletonObject = this.singletonObjects.get(beanName);

    // 一级没有，且这个 Bean 正在创建中，才去查"提前暴露"的缓存
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 二级：已经提前暴露出来的半成品 / 代理
        singletonObject = this.earlySingletonObjects.get(beanName);

        if (singletonObject == null && allowEarlyReference) {
            synchronized (this.singletonObjects) {
                // 双重检查：锁内再看一次，防止并发下重复创建
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    singletonObject = this.earlySingletonObjects.get(beanName);
                    if (singletonObject == null) {
                        // 三级：拿到 ObjectFactory，执行它才有对象
                        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                        if (singletonFactory != null) {
                            singletonObject = singletonFactory.getObject();
                            // 升级到二级，并从三级移除
                            this.earlySingletonObjects.put(beanName, singletonObject);
                            this.singletonFactories.remove(beanName);
                        }
                    }
                }
            }
        }
    }
    return singletonObject;
}
```

这段就是三级缓存的完整逻辑：

- **一级 `singletonObjects`**：完整 Bean，正常 `getBean` 都从这里拿；
- **二级 `earlySingletonObjects`**：提前暴露的半成品（或代理），只有循环依赖时才会出现；
- **三级 `singletonFactories`**：`ObjectFactory`，还没决定要不要生成代理，所以先放"怎么造"的 Lambda；
- **`isSingletonCurrentlyInCreation(beanName)`**：只有正在创建中的 Bean 才允许走"提前暴露"，否则二级/三级根本不会查。

## 创建并登记

`getSingleton(beanName, factory)` 负责"创建 + 标记 + 登记"：

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    synchronized (this.singletonObjects) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            // ① 标记"正在创建"：加入 singletonsCurrentlyInCreation
            //    这样别人来 getSingleton 时，才知道可以走"提前暴露"
            beforeSingletonCreation(beanName);

            boolean newSingleton = false;
            try {
                // ② 真正创建：最终调到 createBean → doCreateBean
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            } catch (IllegalStateException ex) {
                // 循环依赖检测抛出的异常，原样往上抛
                throw ex;
            } finally {
                // ③ 移除"正在创建"标记
                afterSingletonCreation(beanName);
            }

            if (newSingleton) {
                // ④ 创建完成：放入一级缓存，清理二、三级
                addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}
```

`addSingleton` 的作用是把 Bean 放进一级缓存，并清掉二、三级缓存：

```java
protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        this.singletonObjects.put(beanName, singletonObject);
        this.singletonFactories.remove(beanName);     // 三级不再需要
        this.earlySingletonObjects.remove(beanName);  // 二级不再需要
        this.registeredSingletons.add(beanName);
    }
}
```

## addSingletonFactory：把 ObjectFactory 放进三级

`doCreateBean` 在**实例化之后、属性填充之前**，会判断要不要提前暴露：

```java
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
        isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
    // 放进三级缓存的不是对象，而是"怎么拿到对象"的 Lambda
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}
```

为什么必须在属性填充之前？因为 A 填属性时可能要用 B，而 B 又要用 A；此时 A 还没填完，只能先把"怎么拿到 A"的工厂放进去，等 B 来取。

`getEarlyBeanReference` 负责决定"现在要不要生成代理"：

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {
            exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
        }
    }
    return exposedObject;
}
```

`AbstractAutoProxyCreator` 的实现会在这里提前生成代理，并记录到 `earlyProxyReferences`：

```java
@Override
public Object getEarlyBeanReference(Object bean, String beanName) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    this.earlyProxyReferences.put(cacheKey, bean);   // 记下来，避免后面重复代理
    return wrapIfNecessary(bean, beanName, cacheKey); // 需要 AOP 就返回代理，否则返回原对象
}
```

## 把 A ↔ B 走一遍

假设 A、B 都是单例，字段注入，且 A 需要被 AOP 增强：

```text
getBean(A)
  → getSingleton("a")：一级、二级、三级都没有
  → getSingleton("a", factory)
      → beforeSingletonCreation("a")        // a 标记为"正在创建"
      → createBean("a") → doCreateBean("a")
          → createBeanInstance：new A()
          → addSingletonFactory("a", lambda)  // 三级缓存有 a 了
          → populateBean("a")：发现要注入 b
              → getBean(B)
                  → getSingleton("b")：没有
                  → getSingleton("b", factory)
                      → beforeSingletonCreation("b")
                      → createBean("b") → doCreateBean("b")
                          → new B()
                          → addSingletonFactory("b", lambda)
                          → populateBean("b")：发现要注入 a
                              → getBean(A)
                                  → getSingleton("a")
                                     一级没有；isSingletonCurrentlyInCreation("a") = true
                                     二级没有；三级有 a 的 lambda
                                     → 执行 lambda → getEarlyBeanReference("a")
                                     → 需要 AOP：提前生成 a 的代理
                                     → 代理放入二级，三级移除
                                     → 返回 a 的代理
                          → B 拿到 a 的代理
                      → addSingleton("b")    // b 进一级
              → A 拿到 b
          → populateBean("a") 完成
          → initializeBean("a")：此时会看到 earlyProxyReferences 里已有 a，
                                 不再重复生成代理
          → addSingleton("a")       // a（代理）进一级
```

关键点：**B 拿到的是 a 的代理，不是原始对象**。这就是三级缓存里放 `ObjectFactory` 而不是直接放对象的原因——代理是在"第一次有人来提前取"的那一刻才生成的。

## 三级缓存总结

| 缓存 | 放什么 | 写入时机 | 读取时机 |
|---|---|---|---|
| `singletonObjects`（一级） | 完整 Bean（可能是代理） | `addSingleton`，创建完成后 | 每次 `getBean` 先查 |
| `earlySingletonObjects`（二级） | 提前暴露的半成品 / 代理 | 三级 Lambda 执行后升级进来 | 一级没有且正在创建时 |
| `singletonFactories`（三级） | `ObjectFactory`（Lambda） | `doCreateBean` 实例化后、填充属性前 | 二级也没有时才执行 |
| `earlyProxyReferences` | 已提前做过 AOP 的原始对象 | `getEarlyBeanReference` | `postProcessAfterInitialization` 时检查，避免重复代理 |

![img.png](_assets/Totalcycle.png)

![img.png](_assets/example.png)

![img.png](_assets/exampleDetail.png)

# 实战与常见坑

## 场景：Spring Boot 2.6+ 默认禁止循环依赖

Spring Boot 2.6 起，`spring.main.allow-circular-references` 默认是 `false`，出现循环依赖会直接启动失败。需要兼容老代码时可以打开：

```yaml
spring:
  main:
    allow-circular-references: true
```

但更好的做法是重构依赖。实在改不动，可以在其中一个依赖上加 `@Lazy`，让注入的是代理，第一次真正调用时才解析：

```java
@Service
class A {
    @Lazy
    @Autowired
    private B b;
}
```

> `@Lazy` 只是绕过循环依赖，A、B 之间的设计问题还在。

## 常见坑

1. **以为三级缓存能解决构造器注入**：构造器参数在实例化阶段就要解析，此时对象还没进三级缓存，会抛 `BeanCurrentlyInCreationException`。
2. **以为 prototype 也能解**：prototype 不走单例缓存，循环依赖直接报错。
3. **Spring Boot 2.6+ 忘了开配置**：默认禁止循环依赖，老项目升级后可能启动失败。
4. **拿到的是代理却当成原始对象**：循环依赖时 B 注入的是 A 的代理；如果通过其他方式拿到原始对象再操作，会绕过 AOP 增强。
5. **自己往二级缓存塞原始对象**：这会跳过"延迟决定代理"的步骤，导致注入的引用指向原始对象，事务、日志等增强失效。
6. **把三级缓存当成设计目标**：它是兜底，不是鼓励 A、B 互依赖；能通过分层、事件、接口拆分解决的，优先重构。

# 小结

我一向不太喜欢写总结，因为该说的正文里都说了。不过考虑到很多读者是冲着面试来的，还是列成清单：

1. **什么是循环依赖？** A 依赖 B、B 又依赖 A，互相等待，没有任何缓存的实现会无限递归。
2. **一级缓存能解决吗？** 对普通字段注入能——实例化后先放进 Map 再填属性即可。
3. **为什么还要三级？** 因为 AOP：要能在对象未初始化完成时暴露引用，又要保证最终暴露的是**代理**，所以需要工厂来延迟决策。
4. **二级为什么不够？** 二级只能放原始对象，提前注入的引用会指向原始对象，导致代理失效。
5. **三级缓存放的是什么？** `ObjectFactory`（`() -> getEarlyBeanReference(...)`），不是代理对象。
6. **代理是什么时候生成的？** 第一次有人来提前取时，执行三级 Lambda 才生成；没有循环依赖就不会提前生成。
7. **构造器 / prototype 为什么不行？** 构造器注入时对象还没实例化、没得暴露；prototype 不进缓存，而解法依赖缓存。

# 参考

- Spring 5.x 源码：`AbstractBeanFactory.doGetBean`、`DefaultSingletonBeanRegistry`、`AbstractAutowireCapableBeanFactory.getEarlyBeanReference`
- [Spring生命周期.md](Spring生命周期.md)、[SpringAop代理.md](SpringAop代理.md)、[Spring依赖注入.md](Spring依赖注入.md)
