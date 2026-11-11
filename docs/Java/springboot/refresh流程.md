上一篇 [启动流程.md](启动流程.md) 讲的是 `SpringApplication.run` 的外层流程：准备 Environment、创建 Context、执行 Runner。这一篇只讲容器内部的 `refresh()`——它才是 Spring 真正"启动"的地方。

很多人对 `refresh()` 的印象是"一堆钩子方法"，但说不出每一步在做什么、为什么排在这个位置。这篇把十二步逐个拆开，重点回答三个问题：这一步解决什么问题、源码里发生了什么、做完之后容器是什么状态。

> 基于 Spring Boot 2.5.x / Spring Framework 5.3.x。

# refresh 是什么

`refresh()` 是 `AbstractApplicationContext` 的核心方法，负责把"一堆 BeanDefinition"变成"一个可用的容器"。它做了这些事：

- 创建 / 获取 `BeanFactory`；
- 执行 `BeanFactoryPostProcessor`，把 BeanDefinition 补齐；
- 注册 `BeanPostProcessor`，为 Bean 创建做准备；
- 创建 Web 服务器；
- 实例化所有非懒加载单例；
- 启动 Web 服务器、发布容器刷新完成事件。

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
            // 4. 子类扩展：注册 Web 作用域等
            postProcessBeanFactory(beanFactory);

            // 5. 执行 BeanFactoryPostProcessor：自动装配
            invokeBeanFactoryPostProcessors(beanFactory);

            // 6. 注册 BeanPostProcessor
            registerBeanPostProcessors(beanFactory);

            // 7. 初始化国际化
            initMessageSource();

            // 8. 初始化事件广播器
            initApplicationEventMulticaster();

            // 9. 创建 Web 服务器（还没启动）
            onRefresh();

            // 10. 注册监听器
            registerListeners();

            // 11. 实例化所有非懒加载单例
            finishBeanFactoryInitialization(beanFactory);

            // 12. 启动 Web 服务器、发布刷新完成事件
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

`refreshBeanFactory` 在不同容器里的行为不一样，这是很多人混淆的点：

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

# 4. postProcessBeanFactory：子类扩展

这是一个空方法，留给子类定制。Servlet 容器会在这里注册 Web 相关的东西：

```java
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 处理 ServletContextAware
    beanFactory.addBeanPostProcessor(new WebApplicationContextServletContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(ServletContextAware.class);

    // 注册 request / session / application 作用域
    registerWebApplicationScopes();
}
```

`request`、`session`、`application` 这三个作用域不是 Spring 核心自带的，而是 Web 容器在这里注册的。这样后面的 Bean 才能声明成 `@Scope("request")`、`@Scope("session")`。

**做完之后**：Web 作用域可用，Servlet 相关的 Aware 能正常回调。

# 5. invokeBeanFactoryPostProcessors：自动装配发生在这里

这一步执行所有 `BeanFactoryPostProcessor`，顺序是：

1. 先执行 `BeanDefinitionRegistryPostProcessor`，它可以注册新的 BeanDefinition；
2. 再执行普通的 `BeanFactoryPostProcessor`，它只能修改已有的 BeanDefinition。

自动装配的核心 `ConfigurationClassPostProcessor` 就是一个 `BeanDefinitionRegistryPostProcessor`。它在这里：

- 找出所有候选配置类；
- 解析 `@ComponentScan`，扫描出更多 BeanDefinition；
- 解析 `@Import`，包括 `@EnableAutoConfiguration` 引入的 `AutoConfigurationImportSelector`；
- 解析 `@Bean` 方法，注册对应的 BeanDefinition。

这一步结束，**容器里该有的 BeanDefinition 基本都有了**，但还没有 Bean 实例。自动配置的细节见 [starter.md](starter.md)。

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

# 9. onRefresh：创建 Web 服务器

```java
@Override
protected void onRefresh() {
    super.onRefresh();
    createWebServer();
}
```

`createWebServer` 做了几件事：

1. 找到唯一的 `ServletWebServerFactory`（Tomcat / Jetty / Undertow）；
2. 调用 `factory.getWebServer(...)` 创建 `WebServer` 对象；
3. 同时创建 `ServletContext`，并执行所有 `ServletContextInitializer`（比如注册 Filter、Servlet）；
4. 把 `webServerStartStop`、`webServerGracefulShutdown` 两个生命周期 Bean 注册进容器。

注意：**这里只是创建，没有 `start()`**。端口还没有绑定，请求还进不来。

**为什么要在 Bean 实例化之前创建？** 因为有些 Bean 需要拿到 `ServletContext`，比如 `ServletContextAware` 的实现、Filter、Listener。如果等到所有 Bean 创建完再创建服务器，这些 Bean 就没法注入了。

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

**此时 Web 服务器已经创建，但还没启动。** 所以：

- `@PostConstruct` 里可以拿到 `ServletContext`；
- 但访问本机端口会失败，因为 `webServer.start()` 还没调用。

**做完之后**：所有非懒加载单例创建完成，AOP 代理已生成；prototype Bean 不会被创建。

# 12. finishRefresh：启动 Web 服务器

`finishRefresh` 分两层。父类负责生命周期和事件：

```java
protected void finishRefresh() {
    clearResourceCaches();
    initLifecycleProcessor();
    getLifecycleProcessor().onRefresh();

    // 发布容器刷新完成事件
    publishEvent(new ContextRefreshedEvent(this));
}
```

Servlet 子类在父类之后启动服务器：

```java
@Override
protected void finishRefresh() {
    super.finishRefresh();                  // 先发布 ContextRefreshedEvent
    WebServer webServer = startWebServer(); // 再启动服务器
    if (webServer != null) {
        publishEvent(new ServletWebServerInitializedEvent(webServer, this));
    }
}
```

`startWebServer()` 才会真正调用 `webServer.start()`：绑定端口、启动 accept 线程、开始接收请求。

**这里有一个很重要的细节**：`ContextRefreshedEvent` 发布时，Web 服务器还没有启动。如果在这个事件里调用本机接口，一样会失败。真正代表"服务可用"的是外层的 `ApplicationReadyEvent`。

**做完之后**：容器刷新完成，Web 服务器已启动。

# 每一步之后的状态

| 步骤 | BeanDefinition | Bean 实例 | Web 服务器 |
|---|---|---|---|
| 进入 refresh | 主配置类（注解容器） | 无 | 无 |
| `prepareBeanFactory` | 同前 | 基础设施单例 | 无 |
| `invokeBeanFactoryPostProcessors` | 基本齐全 | 无 | 无 |
| `registerBeanPostProcessors` | 同前 | 无 | 无 |
| `onRefresh` | 同前 | 无 | 已创建，未启动 |
| `finishBeanFactoryInitialization` | 同前 | 单例创建完成 | 已创建，未启动 |
| `finishRefresh` | 同前 | 同前 | 已启动 |

# Web 服务器：创建和启动是两回事

**创建**在 `onRefresh`：

```java
private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = getServletContext();

    if (webServer == null && servletContext == null) {
        ServletWebServerFactory factory = getWebServerFactory();
        this.webServer = factory.getWebServer(getSelfInitializer());
        // 注册生命周期 Bean
        getBeanFactory().registerSingleton("webServerStartStop",
                new WebServerStartStopLifecycle(this, this.webServer));
    }
    // ...
}
```

**启动**在 `finishRefresh`：

```java
private WebServer startWebServer() {
    WebServer webServer = this.webServer;
    if (webServer != null) {
        webServer.start();
    }
    return webServer;
}
```

为什么这样拆？因为：

- 创建在前：Bean 实例化时可能需要 `ServletContext`；
- 启动在后：要等所有 Bean 都初始化完，才能对外接收请求。

# 事件顺序

| 事件 / 回调 | 触发位置 | Web 服务器是否启动 |
|---|---|---|
| `@PostConstruct` | `finishBeanFactoryInitialization` | 否 |
| `ContextRefreshedEvent` | `finishRefresh` 的父类部分 | 否 |
| `ApplicationStartedEvent` | `refresh()` 返回后、Runner 前 | 是 |
| `ApplicationReadyEvent` | Runner 执行完后 | 是 |

所以要在启动后调用本服务接口，应该用 `ApplicationRunner` 或监听 `ApplicationReadyEvent`，不要用 `@PostConstruct` 或 `ContextRefreshedEvent`。

# 常见误解

1. **`refresh()` 可以调多次**：`GenericApplicationContext` 第二次调用会直接抛异常；XML 容器虽然能重新 refresh，但会销毁旧容器，业务上不推荐。
2. **`ContextRefreshedEvent` 代表服务可用**：不代表，此时 Web 服务器还没启动；`ApplicationReadyEvent` 才是。
3. **`BeanFactoryPostProcessor` 和 `BeanPostProcessor` 是一回事**：前者改 BeanDefinition，后者改 Bean 实例，时机和对象都不同。
4. **`@PostConstruct` 里能访问本机端口**：不能，服务器还没 `start()`。
5. **`finishBeanFactoryInitialization` 会创建所有 Bean**：只创建非懒加载的单例；prototype 和懒加载 Bean 不会在这里创建。
6. **`onRefresh` 会启动服务器**：只创建，不启动；启动在 `finishRefresh`。

# 小结

我一向不太喜欢写总结，因为该说的正文里都说了。不过考虑到很多读者是冲着面试来的，还是列成清单：

1. **`refresh()` 做了什么？** 准备 BeanFactory、执行 BeanFactoryPostProcessor、注册 BeanPostProcessor、创建 WebServer、实例化 Bean、启动 WebServer。
2. **`obtainFreshBeanFactory` 在注解容器和 XML 容器里有什么区别？** 注解容器的 BeanFactory 在构造时已创建，这里只设序列化 ID；XML 容器会新建 BeanFactory 并加载 BeanDefinition。
3. **自动装配在哪一步？** `invokeBeanFactoryPostProcessors` → `ConfigurationClassPostProcessor`。
4. **为什么 `BeanDefinitionRegistryPostProcessor` 先执行？** 它负责注册新的 BeanDefinition，普通的 `BeanFactoryPostProcessor` 只能修改已有的。
5. **WebServer 什么时候创建、什么时候启动？** `onRefresh` 创建，`finishRefresh` 启动。
6. **`@PostConstruct` 时 WebServer 启动了吗？** 没有，只创建了；`@PostConstruct` 在 `finishBeanFactoryInitialization` 里执行。
7. **`ContextRefreshedEvent` 时服务可用了吗？** 没有，服务器还没启动；`ApplicationReadyEvent` 才是真正可用。
8. **`finishBeanFactoryInitialization` 创建哪些 Bean？** 只创建非懒加载的单例。

# 参考

- Spring Framework 5.3.x 源码：`AbstractApplicationContext.refresh`、`PostProcessorRegistrationDelegate`
- Spring Boot 2.5.x 源码：`ServletWebServerApplicationContext`
- [启动流程.md](启动流程.md)（外层流程）、[starter.md](starter.md)（自动装配）
