Bean 的生命周期是最容易被"背步骤"糊弄过去的部分：大家都能说"实例化、属性填充、初始化"，但再问一句"这行代码在哪、谁调的、为什么排在这个位置"，就卡住了。

这篇不列提纲，直接跟着 `doCreateBean` 的源码走一遍：看一个 Bean 从 BeanDefinition（类的定义信息）到最终可用的对象，具体经过哪些方法，以及每个扩展点为什么排在那个位置。如果只为面试，可以直接滑到最后的**小结**。

> 基于 Spring 5.3.13。主线走注解版（`AnnotationConfigApplicationContext`），XML 形式从略。

# 总览

先看一个最普通的启动入口：

```java
public class App {
    @Test
    public void testApplication() {
        AnnotationConfigApplicationContext context =
                new AnnotationConfigApplicationContext("com.carl.service");   // 指定扫描包
        StudentService studentService = (StudentService) context.getBean("studentService");
        System.out.println(studentService);
    }
}
```

这段代码背后，容器实际做了这么一串事：

```text
扫描包 → 生成 BeanDefinition
  → refresh()
      → invokeBeanFactoryPostProcessors    // 修改 BeanDefinition
      → registerBeanPostProcessors         // 注册 BeanPostProcessor
      → preInstantiateSingletons           // 按 BeanDefinition 创建 Bean
          → getBean → doGetBean
              → getSingleton（查三级缓存）
              → createBean
                  → doCreateBean
                      实例化 → 收集注入点 → 提前暴露 → 属性填充 → 初始化（含 Aware）→ 生成代理
  → 容器正常使用
  → close()：销毁 DisposableBean
```

![img.png](_assets/beanLifeCycle.png)

记住两个阶段就够了：**先把类解析成 BeanDefinition，再根据 BeanDefinition 创建 Bean 实例**。下面分开看。

## preInstantiateSingletons：遍历定义，触发创建

容器启动到最后，`DefaultListableBeanFactory.preInstantiateSingletons()` 会遍历所有**非懒加载的单例**：

```java
@Override
public void preInstantiateSingletons() throws BeansException {
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);   // 所有定义的名字
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 只处理：非抽象、单例、非懒加载
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                // FactoryBean：先取 & 再 getObject()，细节略
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof FactoryBean) {
                    // 只有 SmartFactoryBean 且 isEagerInit() 才继续 getBean
                }
            } else {
                getBean(beanName);   // 核心入口：→ doGetBean → getSingleton → createBean
            }
        }
    }
    // 全部实例化后，回调 SmartInitializingSingleton.afterSingletonsInstantiated()
}
```

注意：这个方法本身**不创建对象**，它只是遍历定义、触发 `getBean`。真正创建在 `getBean → doGetBean` 里：先查缓存（三级缓存、循环依赖都在那里），缓存没有再 `createBean`。缓存细节见 [Spring循环依赖.md](Spring循环依赖.md)。

# 阶段一：先有 BeanDefinition

Spring 不会拿着类名直接 `new`，它先把每个要管理的类解析成一份 `BeanDefinition`，把类型、作用域、依赖、属性、初始化/销毁方法都记在里面。

`BeanDefinition` 由 `ClassPathBeanDefinitionScanner.doScan` 生成：

1. 扫描 `basePackages` 下的 `.class` 文件，拿到 `Resource` 数组；
2. 用 `MetadataReader`（默认 `SimpleMetadataReader`）读每个类的元数据；
3. 用 `excludeFilters` / `includeFilters` 和 `@Conditional` 筛掉不满足条件的类；
4. 给通过的类生成 `ScannedGenericBeanDefinition`；
5. 把定义注册进 `BeanDefinitionRegistry`。

这一步结束时，**容器里只有 BeanDefinition，没有任何 Bean 实例**。这也解释了为什么 `BeanFactoryPostProcessor` 能改 Bean：它执行时对象还没创建，改的就是 BeanDefinition。

```java
public interface BeanFactoryPostProcessor {
    // 在 Bean 实例化之前执行，可以修改所有 BeanDefinition
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

# 阶段二：refresh() 把定义变成对象

## refresh() 里和 Bean 相关的几步

`AbstractApplicationContext.refresh()` 是容器启动的总入口。和 Bean 生命周期相关的，主要是这几步（顺序不能乱）：

1. **`invokeBeanFactoryPostProcessors(beanFactory)`**：在实例化之前执行所有 `BeanFactoryPostProcessor`，改 BeanDefinition；
2. **`registerBeanPostProcessors(beanFactory)`**：把 `BeanPostProcessor` 注册进容器。必须赶在实例化之前，因为后面 `@Autowired` 注入、AOP 代理都依赖它们；
3. **`finishBeanFactoryInitialization(beanFactory)`**：实例化所有非懒加载的单例，最终落到 `preInstantiateSingletons()`。

一句话理解顺序：**先修改 BeanDefinition，再注册 BeanPostProcessor，最后创建 Bean 实例。** 前两步如果放到创建之后，就赶不上这次创建了。

## createBean：给 BeanPostProcessor 一次"提前返回"的机会

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
        implements AutowireCapableBeanFactory {

    @Override
    protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
        // ① 给 BeanPostProcessor 一次机会：直接返回一个成品/代理，跳过正常创建
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;                                  // 短路
        }
        // ② 正常创建
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        return beanInstance;
    }

    protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
        Object bean = null;
        if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
            if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
                Class<?> targetType = determineTargetType(beanName, mbd);
                if (targetType != null) {
                    // 实例化前回调：这里可能直接返回一个对象（例如自定义 TargetSource 的代理）
                    bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                    if (bean != null) {
                        // 已经提前返回了，就不再走 doCreateBean，
                        // 但仍要补一次"初始化后"处理（注意：不是"实例化后"）
                        bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                    }
                }
            }
            mbd.beforeInstantiationResolved = (bean != null);
        }
        return bean;
    }
}
```

大部分 Bean 在这里拿到的是 `null`，所以会继续走 `doCreateBean`。只有自定义 `TargetSource` 这类场景才会提前返回代理，详见 [SpringAop代理.md](SpringAop代理.md)。

> 这里容易看错：提前返回时补的是 `applyBeanPostProcessorsAfterInitialization`，不是"AfterInstantiation"。因为对象已经由 `BeforeInstantiation` 直接给出了，正常的"实例化 + 初始化"被跳过了，只补最后一道后置处理。

## doCreateBean：Bean 的一生

`doCreateBean` 是创建 Bean 的主方法。先整段看，再逐段解释：

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {

    // ① 实例化：先造一个"空壳"，属性还没填
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 反射调构造 / 工厂方法 / Supplier，具体走 createBeanInstance
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();

    // ② 收集注入点：@Autowired 等在这里被扫出来
    //    （AutowiredAnnotationBeanPostProcessor.postProcessMergedBeanDefinition）
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            mbd.postProcessed = true;
        }
    }

    // ③ 提前暴露：把 ObjectFactory 放进三级缓存，循环依赖靠它
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // ④ 属性填充 + 初始化
    Object exposedObject = bean;
    try {
        populateBean(beanName, mbd, instanceWrapper);                  // @Autowired / @Value 注入
        exposedObject = initializeBean(beanName, exposedObject, mbd);  // Aware → 初始化 → AOP 代理
    } catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Initialization of bean failed", ex);
    }

    // ⑤ 注册销毁回调（DisposableBean / destroy-method / @PreDestroy）
    registerDisposableBeanIfNecessary(beanName, bean, mbd);

    return exposedObject;
}
```

主线：**实例化 → 收集注入点 → 提前暴露 → 属性填充 → 初始化（生成代理）→ 注册销毁**。

为什么是这个顺序，逐个说：

- **实例化**：先有对象才能填属性。`createBeanInstance` 里三选一——`Supplier.get()`、`factoryMethod`、默认无参构造。
- **收集注入点**：在填属性之前，先把"哪些字段/方法要注入"扫出来，缓存在 `InjectionMetadata` 里；真正注入在下一步。
- **提前暴露**：只对"单例 + 正在创建 + 允许循环依赖"的 Bean 做。它必须在属性填充**之前**——因为 A 填属性时可能要用 B，而 B 又要用 A，此时 A 还没填完，只能先把"怎么拿到 A"的工厂放进三级缓存。
- **属性填充**：`populateBean` 执行注入，必须有依赖才能继续初始化。
- **初始化**：`initializeBean` 里先回调 Aware，再跑初始化方法，最后生成 AOP 代理。代理必须在**初始化完成后**生成，否则代理包住的是一个属性没填完的半成品。
- **注册销毁**：把需要销毁回调的 Bean 记到 `disposableBeans`，等容器关闭再统一销毁（不是现在销毁）。

## initializeBean：Aware、初始化和代理

`doCreateBean` 的第 ④ 步调用了 `initializeBean`，Bean 的"初始化阶段"全在这里：

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
    // ① 回调 Aware：BeanNameAware / BeanClassLoaderAware / BeanFactoryAware
    invokeAwareMethods(beanName, bean);

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // ② 初始化前：BeanPostProcessor.postProcessBeforeInitialization
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    // ③ 初始化：InitializingBean.afterPropertiesSet() / init-method
    invokeInitMethods(beanName, wrappedBean, mbd);

    if (mbd == null || !mbd.isSynthetic()) {
        // ④ 初始化后：BeanPostProcessor.postProcessAfterInitialization
        //    AOP 代理就是在这里把原对象包成代理返回的
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```

这段的顺序也有讲究：

- **Aware 先回调**：初始化方法里可能要用到 `beanName`、`BeanFactory`，得先给；
- **初始化前**：`@PostConstruct`、`ApplicationContextAware` 等都在这一步；
- **初始化**：`afterPropertiesSet()` 和 `init-method`，此时依赖已经注入完成；
- **初始化后**：AOP 代理在这里生成，所以代理对象是最后一步才替换掉原对象的。

> `invokeAwareMethods` 只处理 `BeanNameAware`、`BeanClassLoaderAware`、`BeanFactoryAware`；`ApplicationContextAware` 等由 `ApplicationContextAwareProcessor` 在"初始化前"回调，详见 [SpringAware.md](SpringAware.md)。

## populateBean：@Autowired 到底在哪一步注入

`initializeBean` 之前，`doCreateBean` 先调了 `populateBean`。它除了处理 XML 的 `<property>`，还会触发 `InstantiationAwareBeanPostProcessor.postProcessProperties`——`@Autowired` 的注入就在这一步：

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    // 实例化之后、属性填充之前的扩展点（返回 false 表示跳过后续填充）
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                return;
            }
        }
    }

    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    // 老式 autowire="byName / byType" 的处理（注解版基本不用）
    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        // autowireByName / autowireByType，略
    }

    // 关键：@Autowired / @Resource / @Value 通过 postProcessProperties 注入
    for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
        PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
        if (pvsToUse == null) {
            return;
        }
        pvs = pvsToUse;
    }

    if (pvs != null) {
        applyPropertyValues(beanName, mbd, bw, pvs);   // 把属性值真正 set 进去
    }
}
```

为什么属性填充必须在初始化之前？因为 `init-method`、`afterPropertiesSet()`、`@PostConstruct` 里经常直接用注入进来的依赖。先初始化再注入，这些方法里拿到的就是 `null`。

> 这和 [Spring依赖注入.md](Spring依赖注入.md) 是一件事的两个视角：那边讲"怎么找、怎么塞"，这里只交代"它发生在生命周期的哪一步"。

# 初始化的三种回调

一个 Bean 的"初始化"有三种写法，触发顺序是固定的：

1. **`@PostConstruct`**：由 `InitDestroyAnnotationBeanPostProcessor`（一个 `BeanPostProcessor`）在"初始化前"阶段调用；
2. **`afterPropertiesSet()`**：实现了 `InitializingBean`，由 `invokeInitMethods` 在"初始化"阶段调用；
3. **`init-method`**：BeanDefinition 里指定的方法，同样在 `invokeInitMethods` 里，排在 `afterPropertiesSet()` 之后。

所以顺序是：`@PostConstruct` → `afterPropertiesSet()` → `init-method`。三者可以同时存在，按上面的顺序依次执行。

# Bean 的销毁

这里要区分两件事：**登记**发生在初始化之后，**真正销毁**发生在容器 `close()` 时。

初始化完成后，Spring 会判断这个 Bean 关闭时要不要回调销毁方法。满足**任意一条**即算：

1. 实现了 `DisposableBean` 或 `AutoCloseable`；
2. BeanDefinition 指定了 `destroyMethod`；
3. `DestructionAwareBeanPostProcessor.requiresDestruction(bean)` 返回 true，其中：
   - `ApplicationListenerDetector` 让 `ApplicationListener` 成为 `DisposableBean`；
   - `InitDestroyAnnotationBeanPostProcessor` 让带 `@PreDestroy` 的方法成为 `DisposableBean`。

符合条件的 Bean 会被适配成 `DisposableBeanAdapter`，存进 `disposableBeans`（一个 `LinkedHashMap`）。等容器调用 `close()` → `destroySingletons()` 时，再按登记顺序统一回调。

# 实战与常见坑

## 场景：三种初始化回调怎么选

`initializeBean` 的顺序是：Aware → `postProcessBeforeInitialization` → `invokeInitMethods` → `postProcessAfterInitialization`。三种初始化写法分别落在不同位置：

| 写法 | 由谁触发 | 适合场景 |
|---|---|---|
| `@PostConstruct` | `InitDestroyAnnotationBeanPostProcessor.postProcessBeforeInitialization` | 业务代码里做轻量初始化，依赖已经注入 |
| `InitializingBean.afterPropertiesSet()` | `invokeInitMethods` | 写基础设施、需要强制契约时 |
| `init-method` | `invokeInitMethods`，排在 `afterPropertiesSet()` 之后 | 想把初始化方法配置化，比如 `@Bean(initMethod = "start")` |

选择建议：业务代码优先 `@PostConstruct`；框架扩展用 `InitializingBean`；需要配置化时用 `init-method`。三者的顺序固定，不要写重复逻辑。

## 常见坑

1. **在构造器里用 `@Autowired` 字段**：构造器执行时属性还没填充，字段是 `null`。需要依赖就放到构造器参数里。
2. **在 `@PostConstruct` 里做耗时操作**：它发生在容器启动阶段，会拖慢启动；耗时逻辑交给 `SmartInitializingSingleton` 或 `ApplicationRunner`。
3. **同时实现 `@PostConstruct` 和 `afterPropertiesSet`**：两个都会执行，顺序是 `@PostConstruct` → `afterPropertiesSet` → `init-method`。
4. **以为 prototype 也会走销毁回调**：容器只管理单例的销毁；prototype 的清理要自己负责。
5. **`BeanPostProcessor` 里直接注入另一个 `BeanPostProcessor`**：容易触发过早初始化，必要时用 `static @Bean` 方法或 `ObjectProvider` 延迟获取。

# 小结

1. **BeanDefinition 从哪来？** `ClassPathBeanDefinitionScanner.doScan` 扫描、筛选、生成，此时还没有 Bean 实例。
2. **容器启动做了啥？** `refresh()` → `invokeBeanFactoryPostProcessors`（修改 BeanDefinition）→ `registerBeanPostProcessors`（注册 BeanPostProcessor）→ `preInstantiateSingletons`（创建 Bean 实例）。
3. **Bean 的完整生命周期？** 实例化 → 收集注入点 → 提前暴露（三级缓存）→ 属性填充 → Aware 回调 → 初始化前 → 初始化 → 初始化后（生成代理）→ 使用 → 销毁。
4. **为什么属性填充在初始化之前？** 否则 `@PostConstruct` / `afterPropertiesSet` 里拿不到注入的依赖。
5. **为什么 AOP 代理在初始化之后？** 代理要包住一个属性完整、初始化完成的对象，而不是半成品。
6. **`resolveBeforeInstantiation` 干什么？** 给 `BeanPostProcessor` 一次提前返回对象（代理）的机会；普通 Bean 返回 null，继续走 `doCreateBean`。
7. **三种初始化回调的顺序？** `@PostConstruct` → `afterPropertiesSet()` → `init-method`。
8. **销毁是什么时候发生的？** 初始化后只是**登记**，真正回调在容器 `close()` 时。

# 参考

- Spring 5.x 源码：`AbstractApplicationContext.refresh`、`AbstractAutowireCapableBeanFactory`、`ClassPathBeanDefinitionScanner`
- [SpringIOC.md](SpringIOC.md)、[Spring依赖注入.md](Spring依赖注入.md)、[SpringAop代理.md](SpringAop代理.md)、[SpringAware.md](SpringAware.md)
