所谓循环依赖（Circular Dependency），就是 A 依赖 B、B 又依赖 A，形成闭环。如果容器老老实实"造 A → 发现要 B → 去造 B → 发现要 A → 又去造 A"，就会无限递归下去。

Spring 用**三级缓存**解开了这个环。这篇讲清楚：为什么要三级、每一级放什么、为什么构造器和 prototype 解不了。

# 总览

先给结论，省得绕：

- **字段注入 / setter 注入**的循环依赖：Spring 能解决；
- **构造器注入**的循环依赖：解决不了（对象还没造出来没法提前暴露）；
- **prototype 作用域**的循环依赖：解决不了（不放进缓存）。

而且，解决循环依赖**不是靠"多级缓存"本身**，真正的原因是 **AOP 代理**。下面用一个小 demo 把这件事说透。

# 一级缓存够不够？

先别急着上三级。用一个最简单的容器试试——只有一个 Map，实例化后先放进缓存再填属性：

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
        singleMap.put(beanName, obj);             // 先入缓存（半成品也能被取到）
        for (Field field : obj.getClass().getDeclaredFields()) {   // 再填属性
            field.setAccessible(true);
            Class<?> fieldClass = field.getType();
            String fieldBeanName = fieldClass.getSimpleName().toLowerCase();
            field.set(obj, singleMap.containsKey(fieldBeanName)
                    ? singleMap.get(fieldBeanName)              // 已在缓存里，直接取（解开循环的关键）
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

所以"一级缓存解决不了循环依赖"这个说法**并不准确**——对**普通字段注入**，一个 Map 就够。那 Spring 为什么还要三级？

# 为什么还要三级缓存

![img.png](_assets/beanRecyle.png)

真正的原因是 **AOP 代理**：

- 循环依赖时，Spring 得在 Bean 还没初始化完成时就把引用**提前暴露**给对方；
- 但代理对象本该在**初始化之后**由 `BeanPostProcessor` 生成；
- 只有一个 Map，就无法区分"原始对象"和"最终代理对象"，也没法在正确的时机把代理换进去。

于是才需要分级：一级放成品、二级放半成品、三级放一个"能产出半成品的工厂"。

## 二级缓存为什么不够

两个缓存（成品 + 半成品）——对普通循环依赖确实够。Spring 没用两级，还是因为 AOP：

如果二级缓存里放的是**原始对象**，那提前暴露出去的就是原始对象；等后续代理生成后，别的 Bean 里注入的引用可能仍指向原始对象，导致增强失效。

所以需要第三级缓存：**放的不是对象，而是一个 `ObjectFactory`（Lambda）**，等到真正被依赖时才执行，从而**延迟决定"要不要提前生成代理"**。

## 三级缓存里到底是什么

三级缓存 `singletonFactories` 里是一个 Lambda：

```java
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```

它的作用就是延迟决策：只有真出现循环依赖、有人来取时，才执行这个 Lambda；执行时会走 `getEarlyBeanReference`——如果该 Bean 需要 AOP，就在这里**提前生成代理**并放入二级缓存，否则直接返回原始对象。

```java
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
        isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
    // 注册一个工厂到三级缓存（注意：不是把代理放进去，而是放"怎么造"的 Lambda）
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}
```

几点要澄清，别被绕晕：

- 属性填充仍然发生在**原始对象**上；
- Bean 完整初始化后，才把最终对象（原始对象或代理）放入一级缓存 `singletonObjects`；
- `addSingletonFactory` 只是把工厂注册进三级缓存，**不是**"把代理从二级移到一级"；
- 还有一个缓存 `earlyProxyReferences`，用来记录某个原始对象**是否已经提前做过 AOP**，避免重复代理。

# 为什么这两种解不了

- **构造器循环依赖**：依赖对象必须在构造方法里就初始化好。而三级缓存生效的前提是"**实例化完、属性还没填**"这个中间态能被提前暴露——构造器注入时，对象连实例都没造出来，自然没得暴露。
- **prototype 循环依赖**：prototype 每次都是 `new` 出来的，**不放进缓存**。而循环依赖的解法完全依赖缓存，所以解不了。

# 三级缓存总结

| 缓存 | 放什么 | 说明 |
|---|---|---|
| `singletonObjects`（一级） | 完整生命周期的 Bean | 最终成品 |
| `earlySingletonObjects`（二级） | 未完整生命周期的 Bean | 循环依赖时提前暴露；需要 AOP 则放代理，否则原始对象 |
| `singletonFactories`（三级） | `ObjectFactory`（Lambda） | 延迟决定要不要提前生成代理 |
| `earlyProxyReferences` | 已提前做过 AOP 的原始对象记录 | 避免重复代理 |

![img.png](_assets/Totalcycle.png)

![img.png](_assets/example.png)

![img.png](_assets/exampleDetail.png)

# 小结

1. **什么是循环依赖？** A 依赖 B、B 依赖 A，形成闭环，朴素实现会无限递归。
2. **一级缓存能解决吗？** 对普通字段注入能——先入缓存再填属性即可。所谓"一级缓存解决不了"并不准确。
3. **为什么还要三级？** 因为 AOP：必须在对象未初始化完成时暴露引用，又要保证最终暴露的是**代理**，所以需要工厂来延迟决策。
4. **二级为什么不够？** 二级里放原始对象会让提前注入的引用指向原始对象、代理失效；三级用 `ObjectFactory` 延迟决定是否提前生成代理。
5. **三级缓存放的是什么？** `ObjectFactory`（`() -> getEarlyBeanReference(...)`），不是代理对象。
6. **构造器 / prototype 为什么不行？** 构造器注入时对象还没实例化、没得暴露；prototype 不进缓存，而解法依赖缓存。

# 参考

- Spring 5.x 源码：`DefaultSingletonBeanRegistry`、`AbstractAutowireCapableBeanFactory.getEarlyBeanReference`
- [Spring生命周期.md](Spring生命周期.md)、[SpringAop代理.md](SpringAop代理.md)、[Spring依赖注入.md](Spring依赖注入.md)
