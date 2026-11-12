`refresh()` 是 Spring Framework 的机制，写在 `AbstractApplicationContext` 里。非 Boot 的 Spring 应用——注解式的 `AnnotationConfigApplicationContext`、XML 式的 `ClassPathXmlApplicationContext`——启动时走的都是这套流程。Boot 只是在它上面接了几根线，那些放在 [Boot扩展refresh.md](Boot扩展refresh.md) 里讲。

面试问到 refresh，很少真让人背十二步。被反复问的是几个节点：BeanDefinition 在哪补齐、两类后置处理器差在哪、Bean 什么时候被创建、哪几步是留给子类的。所以下面先给一个骨架，再挑这几个点说。

## 十二步骨架

refresh 从头到尾是这一串动作：

1. prepareRefresh：把容器标成"运行中"，校验必需属性。
2. obtainFreshBeanFactory：拿到 BeanFactory。
3. prepareBeanFactory：注册容器自身的基础设施，比如 `Environment`、`ApplicationContext` 这些，让业务 Bean 能直接注入。
4. postProcessBeanFactory：空方法，留给子类。
5. invokeBeanFactoryPostProcessors：执行 BeanDefinition 层面的后置处理器。
6. registerBeanPostProcessors：注册 Bean 层面的后置处理器。
7. initMessageSource：国际化。
8. initApplicationEventMulticaster：事件广播器。
9. onRefresh：空方法，留给子类。
10. registerListeners：挂监听器。
11. finishBeanFactoryInitialization：实例化所有非懒加载单例。
12. finishRefresh：收尾，触发 Lifecycle，发布 `ContextRefreshedEvent`。

记一条主线就够：前三步准备工厂，第 5、6 步把两类后置处理器就位，第 11 步创建 Bean，第 12 步收尾。第 4、9 步本身是空的，是留给子类的插口。

## 核心点一：BeanDefinition 在第 5 步补齐

第 5 步执行的是 BeanDefinition 层面的后置处理器，顺序有讲究：先跑能注册新定义的（`BeanDefinitionRegistryPostProcessor`），再跑只能改已有定义的（普通 `BeanFactoryPostProcessor`）。

处理 `@Configuration` 的 `ConfigurationClassPostProcessor` 属于前者。`@ComponentScan` 扫出来的类、`@Import` 导进来的自动配置、`@Bean` 方法声明的 Bean，都是在这一步变成 BeanDefinition。这一步走完，容器里"该有的定义基本都有"，但还没创建任何对象。

这也是注解和容器交汇的地方。注解本身不执行，是第 5 步把它们翻译成了 BeanDefinition。第 5 步之前，注解只是一堆标记。

至于为什么"注册"必须先于"修改"，很好理解：注册型的处理器负责往表里加定义，如果先跑修改型的，它可能根本看不到后面才加进来的那些定义。

## 核心点二：两类后置处理器别搞混

`BeanFactoryPostProcessor` 和 `BeanPostProcessor` 名字只差几个字母，作用完全不同。

前者改的是 BeanDefinition，在第 5 步执行，此时连 Bean 都还没有。后者改的是 Bean 实例，在第 6 步注册、等到第 11 步创建 Bean 时才生效。面试里含混地回答这两者，很容易被抓住。

第 6 步要先把后置处理器准备好，是因为第 11 步创建 Bean 时处处都用到它们：属性填充靠处理 `@Autowired` 的那个，初始化回调靠处理 `@PostConstruct` 的那个，AOP 代理靠 `AnnotationAwareAspectJAutoProxyCreator`。注册晚了，Bean 创建时就用不上。

## 核心点三：Bean 是在第 11 步被创建的

第 11 步对每个非懒加载单例调用 `getBean`，真正的创建链路是：实例化 → 属性填充 → Aware 回调 → 初始化前 → 初始化（`@PostConstruct`、`afterPropertiesSet`、`init-method`）→ 初始化后（生成 AOP 代理）。

有一个面试常挖的坑：这一步只创建非懒加载的单例。原型 Bean 和懒加载 Bean 都不会在这里出现，它们要等到第一次被用到。另外这一步开始前会把配置冻结，之后不再允许改动 BeanDefinition。

## 核心点四：哪几步是留给子类的

第 4 步 `postProcessBeanFactory` 和第 9 步 `onRefresh`，在父类里都是空方法。它们存在的意义是卡在两个关键节点之间：

`postProcessBeanFactory` 的时机是——BeanFactory 已经备好，但 BeanDefinition 还没解析。往这里加东西，改动会早于定义解析生效。

`onRefresh` 的时机是——BeanDefinition 已经齐了，但 Bean 还没实例化。需要"Bean 创建时就能用到、但又不能早于定义解析"的准备，放这里正合适。

第 12 步 `finishRefresh` 有点特殊：它在 Spring 里是有默认实现的，不是空方法，只是允许子类覆盖。Boot 用的其实不是覆盖它，后面单独一篇讲。

## 核心点五：为什么只能 refresh 一次

注解式容器的 BeanFactory 在构造时就建好了，`refresh` 只是设了个标记。第二次调用会直接抛异常，这是它有意为之。XML 容器虽然支持重新 refresh，但会先销毁旧工厂再重建，业务上基本没人这么用。

顺带一个容易混淆的差异：`obtainFreshBeanFactory` 在注解容器里几乎是空操作，在 XML 容器里才是真正新建 BeanFactory、加载 XML 定义的地方。

## 收尾那一步做了什么

第 12 步先清理资源缓存，初始化 LifecycleProcessor，然后让它 `onRefresh`——所有实现了 `SmartLifecycle` 的 Bean 会在此时被启动。最后发布 `ContextRefreshedEvent`，标记容器刷新完成。

注意 Lifecycle 和 `ContextRefreshedEvent` 的先后：Lifecycle 的 `start()` 在前，事件在后。Boot 的内嵌 Web 服务器正是卡在这个 Lifecycle 阶段启动的，所以它比 `ContextRefreshedEvent` 更早一步。这一点在 [Boot扩展refresh.md](Boot扩展refresh.md) 里展开。

## 面试速记

- refresh 属于 Spring Framework，不是 Boot 专有。
- BeanDefinition 在第 5 步补齐，主角是 `ConfigurationClassPostProcessor`。
- `BeanFactoryPostProcessor` 改定义、`BeanPostProcessor` 改实例；后者必须在 Bean 实例化前注册好。
- Bean 在第 11 步创建，且只创建非懒加载单例。
- 第 4、9 步是父类的空方法，留给子类；第 12 步有默认实现。
- 注解式容器只能 refresh 一次。

## 参考

- Spring Framework 5.3.x 源码：`AbstractApplicationContext.refresh`、`PostProcessorRegistrationDelegate`
- [Boot扩展refresh.md](Boot扩展refresh.md)：Boot 对 refresh 的扩展（内嵌 Web 服务器）
- [启动流程.md](启动流程.md)：Boot 的注解与 SpringApplication 编排
