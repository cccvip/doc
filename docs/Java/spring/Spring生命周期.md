上一篇 [SpringIOC.md](SpringIOC.md) 讲了 IOC 的整体思路。这一篇把"一个 Bean 从图纸到销毁"的完整链路走一遍——搞懂它，你才知道自定义扩展该往哪个环节插。

> 基于 Spring 5.3.13。以注解版（`AnnotationConfigApplicationContext`）为主线，XML 形式从略。

# 总览

先看一个最普通的入口：

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

这一句话背后，是这么一条链：

```text
扫描包 → 生成 BeanDefinition
  → refresh()：BeanFactoryPostProcessor 改定义、注册 BeanPostProcessor
    → preInstantiateSingletons()：实例化所有非懒加载 Bean
      → createBean → doCreateBean：
        实例化 → 属性填充 → Aware 回调 → 初始化(前后) → 生成代理
  → 容器使用
  → 关闭：DisposableBean 销毁
```

![img.png](_assets/beanLifeCycle.png)

下面分两段看：**先有定义**，**再有 Bean**。

# 第一段：生成 BeanDefinition

`ClassPathBeanDefinitionScanner.doScan` 负责扫描并生成定义：

1. 扫描 `basePackages` 下的 `.class` 文件，得到 `Resource` 数组；
2. 用 `MetadataReader`（默认实现 `SimpleMetadataReader`）解析每个 `Resource`；
3. 用 `excludeFilters` / `includeFilters` 以及 `@Conditional` 条件注解做筛选；
4. 生成 `ScannedGenericBeanDefinition`；
5. 扫描通过的 Bean，加入结果集。

有了定义，就可以进入容器启动。

# 第二段：refresh() 与 Bean 的诞生

## refresh() 的关键几步

`AbstractApplicationContext.refresh()` 是容器启动的总入口，和 Bean 生命周期相关的主要是这几步：

1. **`invokeBeanFactoryPostProcessors`**：在 Bean 实例化**之前**，执行所有 `BeanFactoryPostProcessor`——此时 BeanDefinition 已就绪，可以改它；
2. 在 BeanFactory 中**注册 `BeanPostProcessor`**；
3. **`finishBeanFactoryInitialization`**：实例化所有**非懒加载**的 Bean；
4. 核心落地在 `DefaultListableBeanFactory.preInstantiateSingletons()`。

## preInstantiateSingletons

容器启动时，`DefaultListableBeanFactory.preInstantiateSingletons()` 会把所有**非懒加载的单例**挨个创建出来：

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
                    // ...
                }
            } else {
                getBean(beanName);   // 核心入口：→ doGetBean → getSingleton → createBean
            }
        }
    }
    // 全部实例化后，回调 SmartInitializingSingleton.afterSingletonsInstantiated()
}
```

这里只是"挨个点名"，真正的创建在 `getBean` → `doGetBean` 里：先查缓存（三级缓存、循环依赖都在那里），没有再 `createBean`。缓存部分见 [Spring循环依赖.md](Spring循环依赖.md)。

## createBean：给 BeanPostProcessor 一次"提前返回"的机会

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
        implements AutowireCapableBeanFactory {

    @Override
    protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
        // ① 给 BeanPostProcessor 一个机会，直接返回一个代理，而不是目标 Bean
        //    （普通 Bean 不走这里，走这里的是自定义 TargetSource 等场景，见 SpringAop代理.md）
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;                                  // 短路，直接返回
        }
        // ② 正常创建 Bean
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        return beanInstance;
    }

    protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
        Object bean = null;
        if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
            if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
                Class<?> targetType = determineTargetType(beanName, mbd);
                if (targetType != null) {
                    // 实例化前回调，可能直接返回一个实例
                    bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                    if (bean != null) {
                        // 若提前返回了，则只执行"初始化后"回调（不是"实例化后"），流程到此为止
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

## doCreateBean：Bean 的一生

真正创建 Bean 的是 `doCreateBean`。先整段贴出来，逐行看：

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

主线记成一句话：**实例化 → 收集注入点 → 提前暴露 → 属性填充 → 初始化（生成代理）→ 注册销毁**。

- **实例化**的三种方式：`Supplier.get()`、`factoryMethod`、默认无参构造，都在 `createBeanInstance` 里选；
- **收集注入点**：`MergedBeanDefinitionPostProcessor`，`@Autowired` 在这里被扫出来；
- **提前暴露**：只对"单例 + 正在创建 + 允许循环依赖"的 Bean 做，详见 [Spring循环依赖.md](Spring循环依赖.md)；
- **属性填充**：下一节 `populateBean`；
- **初始化**：下一节 `initializeBean`；
- **注册销毁**：`registerDisposableBeanIfNecessary`，容器关闭时统一回调。

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

> `invokeAwareMethods` 只处理 `BeanNameAware`、`BeanClassLoaderAware`、`BeanFactoryAware`；`ApplicationContextAware` 等由 `ApplicationContextAwareProcessor` 在"初始化前"回调，详见 [SpringAware.md](SpringAware.md)。

## populateBean：@Autowired 是在哪一步注入的

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

> 这和 [Spring依赖注入.md](Spring依赖注入.md) 是一件事的两个视角：那边讲"怎么找、怎么塞"，这里只交代"它发生在生命周期的哪一步"。

# 初始化的三种回调

Bean 的"初始化"有三种写法，执行顺序是：

1. `@PostConstruct` 标注的方法；
2. `afterPropertiesSet()`（实现 `InitializingBean`）；
3. BeanDefinition 里指定的 `init-method`。

三者都能用，路由顺序如上；`@PostConstruct` 由 `InitDestroyAnnotationBeanPostProcessor` 负责。

# Bean 的销毁

初始化完成后，还有一步：判断这个 Bean 是不是 `DisposableBean`（需要销毁回调）。满足**任意一条**即算：

1. 实现了 `DisposableBean` 或 `AutoCloseable`；
2. BeanDefinition 指定了 `destroyMethod`；
3. `DestructionAwareBeanPostProcessor.requiresDestruction(bean)` 返回 true，其中：
   - `ApplicationListenerDetector` 让 `ApplicationListener` 成为 `DisposableBean`；
   - `InitDestroyAnnotationBeanPostProcessor` 让带 `@PreDestroy` 的方法成为 `DisposableBean`。

符合条件的 Bean 会被适配成 `DisposableBeanAdapter`，存进 `disposableBeans`（一个 `LinkedHashMap`），容器关闭时统一销毁。

# 小结

1. **BeanDefinition 从哪来？** `ClassPathBeanDefinitionScanner.doScan` 扫描、筛选、生成。
2. **容器启动做了啥？** `refresh()` → `invokeBeanFactoryPostProcessors`（改定义）→ 注册 `BeanPostProcessor` → `preInstantiateSingletons`（实例化）。
3. **Bean 的完整生命周期？** 实例化 → 收集注入点 → 提前暴露（三级缓存）→ 属性填充 → Aware 回调 → 初始化前 → 初始化 → 初始化后（生成代理）→ 使用 → 销毁。
4. **`resolveBeforeInstantiation` 干什么？** 给 `BeanPostProcessor` 一个提前返回实例（代理）的机会；普通 Bean 通常返回 null，走正常的 `doCreateBean`。
5. **三种初始化回调的顺序？** `@PostConstruct` → `afterPropertiesSet()` → `init-method`。
6. **怎么成为可销毁 Bean？** 实现 `DisposableBean`/`AutoCloseable`、指定 `destroyMethod`、或带 `@PreDestroy` 等。

# 参考

- Spring 5.x 源码：`AbstractApplicationContext.refresh`、`AbstractAutowireCapableBeanFactory`、`ClassPathBeanDefinitionScanner`
- [SpringIOC.md](SpringIOC.md)、[Spring依赖注入.md](Spring依赖注入.md)、[SpringAop代理.md](SpringAop代理.md)、[SpringAware.md](SpringAware.md)
