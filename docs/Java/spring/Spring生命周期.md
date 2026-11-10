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

*分支很多，只抓核心：*

1. 遍历所有 BeanDefinition 的名字；
2. 判断是否已在三级缓存中，不在就执行 `createBean(beanName, mbd, args)`；
3. `createBean` 里先把 `beanClass` 解析成 `Class`（`doResolveBeanClass`）。

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

真正创建 Bean 的是 `doCreateBean`，按顺序：

1. **实例化**，三种方式任选：
   - BeanDefinition 设了 `Supplier` → 调 `Supplier.get()`；
   - 设了 `factoryMethod` → 用工厂方法；
   - 都没有 → 默认用无参构造；
2. **`MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()`**：属性赋值前的扩展点（`@Autowired` 等就是在这里收集注入点的）；
3. **属性填充**：处理 `@Autowired`、`@Resource`、`@Value` 等，由 `InstantiationAwareBeanPostProcessor.postProcessProperties()` 完成（详见 [Spring依赖注入.md](Spring依赖注入.md)）；
4. **Aware 回调**：给 Bean 回传容器相关的对象：
   - `BeanNameAware` → `beanName`
   - `BeanClassLoaderAware` → `classLoader`
   - `BeanFactoryAware` → `beanFactory`
   - 更多 Aware 见 [SpringAware.md](SpringAware.md)
5. **初始化前**：`BeanPostProcessor.postProcessBeforeInitialization()`；
6. **初始化**：
   - 实现了 `InitializingBean` → 调 `afterPropertiesSet()`；
   - BeanDefinition 指定的 `init-method`；
7. **初始化后**：`BeanPostProcessor.postProcessAfterInitialization()`——**AOP 代理在这里生成**（见 [SpringAop代理.md](SpringAop代理.md)）。

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
3. **Bean 的完整生命周期？** 实例化 → 属性填充 → Aware → 初始化前 → 初始化 → 初始化后（生成代理）→ 使用 → 销毁。
4. **`resolveBeforeInstantiation` 干什么？** 给 `BeanPostProcessor` 一个提前返回实例（代理）的机会；普通 Bean 通常返回 null，走正常的 `doCreateBean`。
5. **三种初始化回调的顺序？** `@PostConstruct` → `afterPropertiesSet()` → `init-method`。
6. **怎么成为可销毁 Bean？** 实现 `DisposableBean`/`AutoCloseable`、指定 `destroyMethod`、或带 `@PreDestroy` 等。

# 参考

- Spring 5.x 源码：`AbstractApplicationContext.refresh`、`AbstractAutowireCapableBeanFactory`、`ClassPathBeanDefinitionScanner`
- [SpringIOC.md](SpringIOC.md)、[Spring依赖注入.md](Spring依赖注入.md)、[SpringAop代理.md](SpringAop代理.md)、[SpringAware.md](SpringAware.md)
