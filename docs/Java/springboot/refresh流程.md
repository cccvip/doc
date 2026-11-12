`refresh()` 是 **Spring Framework** 的机制，属于 `AbstractApplicationContext`，跟 Boot 无关。非 Boot 的应用——`AnnotationConfigApplicationContext`、XML 的 `ClassPathXmlApplicationContext`——走的也是同一套十二步。

Boot 只是在这套流程上做扩展：覆盖 `postProcessBeanFactory`、`onRefresh` 等模板方法，再注册一个 `WebServerStartStopLifecycle`（`SmartLifecycle`）来启动内嵌 Web 服务器，另外加上注解驱动的自动装配。这些 Boot 特有的部分单独写在 [Boot扩展refresh.md](Boot扩展refresh.md)，本篇只讲 Spring 的通用流程。

> 基于 Spring Framework 5.3.x。

# refresh 是什么

`refresh()` 负责把"一堆 BeanDefinition"变成"一个可用的容器"。它做的事：

- 创建 / 获取 `BeanFactory`；
- 执行 `BeanFactoryPostProcessor`，把 BeanDefinition 补齐；
- 注册 `BeanPostProcessor`，为创建 Bean 做准备；
- 初始化国际化与事件广播器；
- 实例化所有非懒加载单例；
- 收尾、发布容器刷新完成事件。

它通常只能调用一次。对 `GenericApplicationContext`（注解容器）来说，第二次调用会直接抛异常；对 XML 容器来说，虽然可以重新 refresh，但会销毁旧容器重建，业务上很少这么用。

# 全貌

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 1. 启动前准备
        prepareRefresh();

        // 2. 创建 / 获取 BeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 3. 给 BeanFactory 注册基础设施
        prepareBeanFactory(beanFactory);

        try {
            // 4. 子类扩展（模板方法）
            postProcessBeanFactory(beanFactory);

            // 5. 执行 BeanFactoryPostProcessor
            invokeBeanFactoryPostProcessors(beanFactory);

            // 6. 注册 BeanPostProcessor
            registerBeanPostProcessors(beanFactory);

            // 7. 初始化国际化
            initMessageSource();

            // 8. 初始化事件广播器
            initApplicationEventMulticaster();

            // 9. 子类扩展：刷新前的额外工作
            onRefresh();

            // 10. 注册监听器
            registerListeners();

            // 11. 实例化所有非懒加载单例
            finishBeanFactoryInitialization(beanFactory);

            // 12. 收尾、发布刷新完成事件
            finishRefresh();
        }
        catch (BeansException ex) {
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        }
        finally {
            resetCommonCaches();
        }
    }
}
```

注意第 4、9、12 步都是**留给子类的扩展点**：Spring 本身几乎不做事，具体行为由子类决定。Boot 就是在这里插进去的。

下面按顺序拆。

# 1. prepareRefresh：启动前准备

```java
protected void prepareRefresh() {
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);

    // 子类可以初始化额外的 PropertySource
    initPropertySources();

    // 校验 @Required 一类标记的必需属性
    getEnvironment().validateRequiredProperties();

    // 把提前注册的监听器放进正式列表
    this.applicationListeners.clear();
    this.applicationListeners.addAll(this.earlyListeners);
}
```

这一步不创建任何 Bean，只做状态准备：

- 记录启动时间，后面统计启动耗时用；
- 把 `closed` 置为 false、`active` 置为 true，表示容器正在运行；
- `initPropertySources` 给子类一个机会补充 PropertySource；
- 校验必需属性，缺了直接启动失败；
- 把 `earlyListeners` 里的监听器转移到正式列表。

**做完之后**：容器状态就绪，但 BeanFactory 和 Bean 都还没有。

# 2. obtainFreshBeanFactory：拿到 BeanFactory

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    return getBeanFactory();
}
```

`refreshBeanFactory` 在不同容器里的行为不一样：

**注解容器（`GenericApplicationContext`）**：BeanFactory 在容器构造时就已经创建好了，`refreshBeanFactory` 只是设置序列化 ID，并保证只 refresh 一次：

```java
@Override
protected final void refreshBeanFactory() throws IllegalStateException {
    if (!this.refreshed.compareAndSet(false, true)) {
        throw new IllegalStateException(
                "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
    }
    this.beanFactory.setSerializationId(getId());
}
```

**XML 容器（`AbstractRefreshableApplicationContext`）**：会销毁旧的 BeanFactory，新建一个 `DefaultListableBeanFactory`，然后加载 XML 里的 BeanDefinition：

```java
@Override
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    beanFactory.setSerializationId(getId());
    loadBeanDefinitions(beanFactory);
    this.beanFactory = beanFactory;
}
```

**做完之后**：`BeanFactory` 可用。注解容器里此时只有主配置类一个 BeanDefinition；XML 容器里则已经加载了 XML 定义的 BeanDefinition。

# 3. prepareBeanFactory：注册基础设施

这一步往 BeanFactory 里注册一批"容器自身"的东西，让后面的 Bean 可以直接注入它们，而不用用户自己定义。

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 类加载器、表达式解析器、属性编辑器
    beanFactory.setBeanClassLoader(getClassLoader());
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // 处理 Aware 回调
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    // ...

    // 让 Bean 可以直接注入容器相关对象
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // 识别 ApplicationListener
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // 注册默认的环境 Bean
    beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
}
```

几个关键点：

- `ApplicationContextAwareProcessor` 负责回调 `ApplicationContextAware`、`EnvironmentAware` 等 Aware 接口；
- `registerResolvableDependency` 让 `@Autowired ApplicationContext`、`@Autowired BeanFactory` 这类注入能成功；
- `environment`、`systemProperties`、`systemEnvironment` 被注册成单例，所以可以直接按名字注入；
- `ApplicationListenerDetector` 负责把 `ApplicationListener` Bean 登记到事件广播器。

**做完之后**：BeanFactory 有了基础设施，但还没有业务 Bean。

# 4. postProcessBeanFactory：子类扩展点

`AbstractApplicationContext` 里这是一个**空方法**：

```java
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 留给子类：BeanFactory 已准备好、但 BeanFactoryPostProcessor 还没执行
}
```

时机很关键：它紧跟在 `prepareBeanFactory` 之后、`invokeBeanFactoryPostProcessors` 之前。也就是说，子类可以在这里往 BeanFactory 里加东西，而这些改动会**早于** BeanDefinition 的解析生效。

Boot 的 Servlet 容器就在这里注册 `request` / `session` / `application` 作用域，详见 [Boot扩展refresh.md](Boot扩展refresh.md)。

**做完之后**：子类要加的基础设施已就位。

# 5. invokeBeanFactoryPostProcessors：BeanDefinition 补齐

这一步执行所有 `BeanFactoryPostProcessor`，顺序是：

1. 先执行 `BeanDefinitionRegistryPostProcessor`，它可以注册新的 BeanDefinition；
2. 再执行普通的 `BeanFactoryPostProcessor`，它只能修改已有的 BeanDefinition。

处理 `@Configuration` 的 `ConfigurationClassPostProcessor` 就是一个 `BeanDefinitionRegistryPostProcessor`。它在这里：

- 找出所有候选配置类；
- 解析 `@ComponentScan`，扫描出更多 BeanDefinition；
- 解析 `@Import`，包括 `@EnableAutoConfiguration` 引入的 `AutoConfigurationImportSelector`；
- 解析 `@Bean` 方法，注册对应的 BeanDefinition。

这一步结束，**容器里该有的 BeanDefinition 基本都有了**，但还没有 Bean 实例。自动配置属于 Boot，细节见 [Boot扩展refresh.md](Boot扩展refresh.md) 和 [starter.md](starter.md)。

**为什么必须是"注册"先于"修改"？** 因为 `BeanDefinitionRegistryPostProcessor` 的职责就是往注册表里加定义；如果先执行普通 `BeanFactoryPostProcessor`，它可能看不到后面才注册进来的定义。

# 6. registerBeanPostProcessors：为创建 Bean 做准备

BeanDefinition 有了，但创建 Bean 还需要 `BeanPostProcessor`，比如：

- `AutowiredAnnotationBeanPostProcessor`：处理 `@Autowired`；
- `CommonAnnotationBeanPostProcessor`：处理 `@PostConstruct`、`@Resource`；
- `AnnotationAwareAspectJAutoProxyCreator`：生成 AOP 代理。

这一步会把这些后置处理器实例化并注册，顺序是：

1. `PriorityOrdered`；
2. `Ordered`；
3. 没有实现排序接口的；
4. 内部的 `MergedBeanDefinitionPostProcessor` 最后再注册一遍。

**为什么要在实例化 Bean 之前做？** 因为后面 `getBean` 的过程中要用到它们：属性填充要靠 `AutowiredAnnotationBeanPostProcessor`，初始化回调要靠 `CommonAnnotationBeanPostProcessor`，AOP 代理要靠 `AnnotationAwareAspectJAutoProxyCreator`。晚了就赶不上。

**做完之后**：后置处理器就绪，但还没有 Bean 实例。

# 7. initMessageSource：国际化

如果容器里没有名为 `messageSource` 的 Bean，就创建一个 `DelegatingMessageSource` 并注册成单例；有的话直接用它。

这一步大多数业务感知不到，但 `MessageSourceAware` 的 Bean 会拿到它。

# 8. initApplicationEventMulticaster：事件广播器

如果容器里没有名为 `applicationEventMulticaster` 的 Bean，就创建一个 `SimpleApplicationEventMulticaster` 并注册成单例。

后面 `publishEvent` 都走这个广播器。默认是同步调用，所以监听器里做耗时操作会拖慢启动。

# 9. onRefresh：子类扩展点

`AbstractApplicationContext` 里这也是一个**空方法**：

```java
protected void onRefresh() throws BeansException {
    // 留给子类：BeanFactoryPostProcessor 已执行完、单例还没实例化
}
```

时机在 `registerBeanPostProcessors` 之后、`finishBeanFactoryInitialization` 之前。

它存在的意义是：**有些工作必须在 Bean 实例化之前完成，但又不能早于 BeanDefinition 解析。** 典型用途就是子类在这里准备"Bean 创建时需要用到的东西"。

Boot 的 Servlet 容器就在这里创建 Web 服务器，之所以放在这个位置，是因为部分 Bean（如 `ServletContextAware`、Filter）需要 `ServletContext`。注意：**这里只创建、不启动**——它同时注册的 `webServerStartStop`、`webServerGracefulShutdown` 两个 `SmartLifecycle` 才负责启动/停机，启动发生在第 12 步。详见 [Boot扩展refresh.md](Boot扩展refresh.md)。

# 10. registerListeners：注册监听器

把容器里的 `ApplicationListener` Bean 注册到事件广播器；同时把之前攒下的 early events 发布出去。

**做完之后**：事件机制就绪。

# 11. finishBeanFactoryInitialization：实例化 Bean

这一步实例化所有**非懒加载的单例**：

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 设置 ConversionService
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) && beanFactory.isTypeMatch(...)) {
        beanFactory.setConversionService(beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // 添加占位符解析器
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // 先实例化 LoadTimeWeaverAware
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // 冻结配置，之后不能再改 BeanDefinition
    beanFactory.setTempClassLoader(null);
    beanFactory.freezeConfiguration();

    // 实例化所有非懒加载单例
    beanFactory.preInstantiateSingletons();
}
```

`preInstantiateSingletons` 会遍历 BeanDefinition，对每个单例调用 `getBean`。`getBean` 的链路是：

```text
getBean → doGetBean → getSingleton → createBean → doCreateBean
```

`doCreateBean` 里依次做：实例化 → 属性填充 → Aware 回调 → 初始化前 → 初始化（`@PostConstruct` / `afterPropertiesSet` / `init-method`）→ 初始化后（AOP 代理）。

**做完之后**：所有非懒加载单例创建完成，AOP 代理已生成。懒加载单例和 prototype Bean 不会被创建。

# 12. finishRefresh：收尾与事件

```java
protected void finishRefresh() {
    // 清理资源缓存
    clearResourceCaches();

    // 初始化生命周期处理器，并触发 onRefresh
    initLifecycleProcessor();
    getLifecycleProcessor().onRefresh();

    // 发布容器刷新完成事件
    publishEvent(new ContextRefreshedEvent(this));
}
```

这一步做三件事：

1. 清理资源缓存；
2. 初始化 `LifecycleProcessor`，对实现了 `Lifecycle` 的 Bean 触发 `start()`；
3. 发布 `ContextRefreshedEvent`，标记容器刷新完成。

注意第 2 件事：`lifecycleProcessor.onRefresh()` 会启动所有 `SmartLifecycle` Bean。**Boot 的内嵌 Web 服务器就是在这里被启动的**——`WebServerStartStopLifecycle.start()` 调用 `webServer.start()` 并发布 `ServletWebServerInitializedEvent`，随后才轮到 `ContextRefreshedEvent`。也就是说，服务器启动**早于** `ContextRefreshedEvent`。详见 [Boot扩展refresh.md](Boot扩展refresh.md)。

**做完之后**：容器刷新完成。

# 每一步之后的状态

| 步骤 | BeanDefinition | Bean 实例 |
|---|---|---|
| 进入 refresh | 主配置类（注解容器） | 无 |
| `prepareBeanFactory` | 同前 | 基础设施单例 |
| `invokeBeanFactoryPostProcessors` | 基本齐全 | 无 |
| `registerBeanPostProcessors` | 同前 | 无 |
| `finishBeanFactoryInitialization` | 同前 | 非懒加载单例创建完成 |
| `finishRefresh` | 同前 | 同前 |

# 常见误解

1. **`refresh()` 是 Boot 的东西**：不是，它是 Spring Framework 的标准流程，非 Boot 应用也用。
2. **`refresh()` 可以调多次**：`GenericApplicationContext` 第二次调用会直接抛异常；XML 容器虽然能重新 refresh，但会销毁旧容器，业务上不推荐。
3. **`BeanFactoryPostProcessor` 和 `BeanPostProcessor` 是一回事**：前者改 BeanDefinition，后者改 Bean 实例，时机和对象都不同。
4. **`postProcessBeanFactory` / `onRefresh` 是"内容方法"**：不是，在 `AbstractApplicationContext` 里它们是空方法，行为全由子类决定。
5. **`finishBeanFactoryInitialization` 会创建所有 Bean**：只创建非懒加载的单例；prototype 和懒加载 Bean 不会在这里创建。

# 小结

我一向不太喜欢写总结，因为该说的正文里都说了。不过考虑到很多读者是冲着面试来的，还是列成清单：

1. **`refresh()` 属于谁？** Spring Framework 的 `AbstractApplicationContext`，不是 Boot 专有。
2. **十二步里哪几步是给子类的？** `postProcessBeanFactory`、`onRefresh` 是空方法（Boot 覆盖它们）；`finishRefresh` 有默认实现、也可覆盖，但 Boot 2.5 不覆盖，改用 `SmartLifecycle` 启动 WebServer。
3. **`obtainFreshBeanFactory` 在注解容器和 XML 容器里有什么区别？** 注解容器的 BeanFactory 在构造时已创建，这里只设序列化 ID；XML 容器会新建 BeanFactory 并加载 BeanDefinition。
4. **BeanDefinition 在哪一步补齐？** `invokeBeanFactoryPostProcessors` → `ConfigurationClassPostProcessor`。
5. **为什么 `BeanDefinitionRegistryPostProcessor` 先执行？** 它负责注册新的 BeanDefinition，普通的 `BeanFactoryPostProcessor` 只能修改已有的。
6. **Bean 在哪一步实例化？** `finishBeanFactoryInitialization`，只创建非懒加载单例。
7. **`ContextRefreshedEvent` 什么时候发布？** `finishRefresh`，标记容器刷新完成。

# 参考

- Spring Framework 5.3.x 源码：`AbstractApplicationContext.refresh`、`PostProcessorRegistrationDelegate`
- [Boot扩展refresh.md](Boot扩展refresh.md)（Boot 的三个覆盖点）、[启动流程.md](启动流程.md)（Boot 的注解与编排）
