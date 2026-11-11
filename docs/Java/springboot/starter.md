Spring Boot 最核心的"魔法"就是自动装配：引入一个依赖，`DataSource`、`RedisTemplate`、`DispatcherServlet` 就自动配好了。很多人能说出 `@EnableAutoConfiguration`，但再问"自动配置类从哪来、条件什么时候判断、多个自动配置谁先谁后、自己写 starter 要注意什么"，就说不清了。

这篇按"启动入口 → 自动配置类清单 → 条件过滤 → 排序 → 注册 → 创建 Bean"的顺序，把自动装配的源码走一遍，最后给一份可落地的 starter 编写规范。如果只为面试，可以直接滑到最后的**小结**。

> 基于 Spring Boot 2.x，同时标注 2.7 / 3.x 的差异。自动配置的运行时消费（`refresh()`、`getBean`）见 [../spring/Spring生命周期.md](../spring/Spring生命周期.md)。

# 总览

**一句话：自动装配 = `@SpringBootApplication` 触发 `@EnableAutoConfiguration`，导入 `AutoConfigurationImportSelector`，读取 classpath 下的自动配置类清单，按条件过滤、排序后注册成 BeanDefinition，最后由 `refresh()` 创建成 Bean。**

```text
SpringApplication.run
  → @SpringBootApplication
      = @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
  → @EnableAutoConfiguration
      = @AutoConfigurationPackage + @Import(AutoConfigurationImportSelector)
  → AutoConfigurationImportSelector（DeferredImportSelector）
      → 读取 META-INF/spring.factories（2.x）
        或 META-INF/spring/...AutoConfiguration.imports（2.7+/3.x）
      → 去重、排除、条件过滤
      → AutoConfigurationSorter 排序（@AutoConfigureBefore/After/Order）
      → 注册成 BeanDefinition
  → refresh() → 实例化 Bean
```

# 核心概念

## @SpringBootApplication 是什么

它是三个注解的组合：

```java
@SpringBootConfiguration   // 本质是 @Configuration
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication {
}
```

- `@SpringBootConfiguration`：标识这是一个配置类；
- `@EnableAutoConfiguration`：开启自动装配；
- `@ComponentScan`：扫描当前包及子包下的 `@Component`。

## @EnableAutoConfiguration 是什么

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
}
```

关键是 `@Import(AutoConfigurationImportSelector.class)`。`@Import` 有四种用法：导入 `@Configuration` 类、`ImportSelector`、`ImportBeanDefinitionRegistrar`、普通组件类。Spring Boot 用的是第二种——`ImportSelector`。

## 自动配置类从哪来

| 版本 | 清单文件 |
|---|---|
| Spring Boot 2.6 及以前 | `META-INF/spring.factories` |
| Spring Boot 2.7 / 3.x | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |

`spring.factories` 的格式：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.AutoConfigA,\
com.example.AutoConfigB
```

`AutoConfiguration.imports` 的格式更简单，一行一个类：

```text
com.example.AutoConfigA
com.example.AutoConfigB
```

> 2.7 开始两种都支持，3.x 自动配置只认 `AutoConfiguration.imports`；`spring.factories` 仍用于其他扩展点（`ApplicationListener` 等）。

## 条件注解

自动配置类不是无条件生效的，常见条件：

| 注解 | 作用 |
|---|---|
| `@ConditionalOnClass` | classpath 上有指定类才生效 |
| `@ConditionalOnMissingClass` | classpath 上没有指定类才生效 |
| `@ConditionalOnBean` | 容器里已有指定 Bean 才生效 |
| `@ConditionalOnMissingBean` | 容器里没有指定 Bean 才生效（最常用） |
| `@ConditionalOnProperty` | 配置项满足条件才生效 |
| `@ConditionalOnWebApplication` | Web 应用才生效 |
| `@ConditionalOnResource` | 指定资源存在才生效 |

## 排除自动配置

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
```

或者配置：

```yaml
spring:
  autoconfigure:
    exclude: org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

# 源码分析

## 入口：SpringApplication.run 概览

```java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return new SpringApplication(primarySource).run(args);
}

public ConfigurableApplicationContext run(String... args) {
    // 1. 准备环境：读取 application.yml、命令行参数、环境变量
    ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);

    // 2. 创建 ApplicationContext（Servlet / Reactive / 普通）
    context = createApplicationContext();

    // 3. 准备上下文：注册 primarySource、加载 BeanDefinition
    prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);

    // 4. 刷新容器：自动装配、实例化 Bean 都在这里
    refreshContext(context);

    // 5. 回调 Runner
    afterRefresh(context, applicationArguments);
    listeners.started(context, timeTakenToStartup);
    listeners.running(context);
    return context;
}
```

自动装配发生在第 4 步 `refreshContext` → `refresh()` → `invokeBeanFactoryPostProcessors` 阶段，具体由 `ConfigurationClassPostProcessor` 解析 `@Import`。

## @Import 的解析：ConfigurationClassPostProcessor

`ConfigurationClassPostProcessor` 是一个 `BeanDefinitionRegistryPostProcessor`（也是 `BeanFactoryPostProcessor`），它在 `refresh()` 的 `invokeBeanFactoryPostProcessors` 阶段执行：

```java
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor, ... {

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        processConfigBeanDefinitions(registry);
    }

    public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
        // 1. 找出所有候选配置类（@Configuration、@Component、@Import 等）
        List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
        for (String beanName : registry.getBeanDefinitionNames()) {
            BeanDefinition beanDef = registry.getBeanDefinition(beanName);
            if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
                configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
            }
        }

        // 2. 解析：@ComponentScan、@Import、@Bean 都在这里处理
        ConfigurationClassParser parser = new ConfigurationClassParser(...);
        Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.parse(candidates));

        // 3. 把解析出来的配置类注册成 BeanDefinition
        this.reader.loadBeanDefinitions(configClasses);
    }
}
```

`@Import(AutoConfigurationImportSelector.class)` 就是在第 2 步的 `processImports` 里被处理的。

## AutoConfigurationImportSelector：自动配置的总入口

```java
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
        ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {

    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        }
        AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
        return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
    }

    protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        }
        AnnotationAttributes attributes = getAttributes(annotationMetadata);

        // ① 读出所有候选自动配置类
        List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
        configurations = removeDuplicates(configurations);

        // ② 去掉 @SpringBootApplication(exclude=...) 和配置里的 exclude
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);

        // ③ 条件过滤：OnClassCondition 等
        configurations = getConfigurationClassFilter().filter(configurations);

        // ④ 发布事件，便于排查
        fireAutoConfigurationImportEvents(configurations, exclusions);
        return new AutoConfigurationEntry(configurations, exclusions);
    }
}
```

## getCandidateConfigurations：读取清单

Spring Boot 2.6 及以前：

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
            getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories...");
    return configurations;
}
```

Spring Boot 2.7 / 3.x：

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = new ArrayList<>(
            ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader()).getCandidates());
    Assert.notEmpty(configurations, "No auto configuration classes found in "
            + "META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports...");
    return configurations;
}
```

这就是"自动配置类从哪来"的答案：**从 classpath 下所有 jar 的清单文件里读出来**，不是靠包扫描。

## 条件过滤：为什么有的自动配置没生效

`getConfigurationClassFilter().filter(configurations)` 会用 `AutoConfigurationImportFilter` 过滤，最核心的是 `OnClassCondition`：

```java
class OnClassCondition extends FilteringSpringBootCondition {

    @Override
    protected ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
            AutoConfigurationMetadata autoConfigurationMetadata) {
        // 解析每个自动配置类上的 @ConditionalOnClass
        // 如果 classpath 上没有指定类，就直接把这个自动配置类过滤掉
        // ...
    }
}
```

除了 `OnClassCondition`，还有 `OnBeanCondition`（`@ConditionalOnBean` / `@ConditionalOnMissingBean`）和 `OnWebApplicationCondition`。这些条件在**注册 BeanDefinition 之前**判断，避免加载一堆用不上的配置。

## 排序：加载顺序由谁决定

自动配置类的顺序由 `AutoConfigurationSorter` 决定，规则是：

1. **`@AutoConfigureOrder`**：绝对顺序，值越小越靠前；
2. **`@AutoConfigureBefore` / `@AutoConfigureAfter`**：相对某个自动配置类的前后；
3. **字母序**：以上都没有时，按类名字典序兜底。

```java
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
public class MyDataSourceAutoConfiguration {
    // ...
}
```

几个要点：

- `@AutoConfigureBefore/After` **只对自动配置类生效**，普通 `@Configuration` 不参与这个排序；
- `@ConditionalOnMissingBean` 只能看到"已经注册的 Bean"，所以顺序很重要：如果 A 要判断 B 是否已定义，A 必须排在 B 后面（用 `@AutoConfigureAfter(B.class)`）；
- `spring.factories` / `AutoConfiguration.imports` 里的书写顺序**不是最终顺序**，最终由 sorter 重排。

## 为什么 DeferredImportSelector 很关键

`AutoConfigurationImportSelector` 实现的是 `DeferredImportSelector`，不是普通 `ImportSelector`。区别在于：

- 普通 `ImportSelector`：在处理当前配置类时**立即**执行；
- `DeferredImportSelector`：等**所有用户配置类**解析完之后，再统一执行。

这保证了：

1. 用户自己的 `@Bean` 先注册，自动配置里的 `@ConditionalOnMissingBean` 才能看到它们；
2. 自动配置类是"兜底"，不会覆盖用户显式定义的 Bean。

`ConfigurationClassParser` 会把 `DeferredImportSelector` 收集起来，最后交给 `DeferredImportSelectorHandler.process()` 统一处理；`AutoConfigurationImportSelector` 内部再用 `AutoConfigurationGroup` 做排序和去重。

# 实战：写一个 starter

## 命名与模块划分

官方 starter 命名是 `spring-boot-starter-xxx`；第三方一般叫 `xxx-spring-boot-starter`。

推荐拆成两个模块：

```text
xxx-spring-boot-autoconfigure   // 自动配置代码、@ConfigurationProperties
xxx-spring-boot-starter         // 只放依赖，把 autoconfigure 和第三方依赖引进来
```

单模块也能用，但拆开后，用户可以只引 autoconfigure、不引全部依赖。

## 自动配置类

```java
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
@ConditionalOnClass(MyClient.class)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyClient myClient(MyProperties properties) {
        return new MyClient(properties.getEndpoint(), properties.getTimeout());
    }
}
```

要点：

- 用 `@AutoConfiguration`（2.7+）或 `@Configuration(proxyBeanMethods = false)`；
- `@ConditionalOnClass` 保证依赖不存在时不会报错；
- `@ConditionalOnMissingBean` 允许用户覆盖你的默认 Bean；
- 自动配置类**不要放在会被 `@ComponentScan` 扫到的包**，否则会绕过条件判断，被当普通配置类加载。

## 配置属性

```java
@ConfigurationProperties(prefix = "my.client")
public class MyProperties {
    private String endpoint = "http://localhost:8080";
    private int timeout = 3000;
    // getter / setter
}
```

配合 `@EnableConfigurationProperties(MyProperties.class)` 或 `@ConfigurationPropertiesScan` 使用。再加 `spring-boot-configuration-processor` 依赖，IDE 会有配置提示。

## 注册自动配置

Spring Boot 2.6 及以前，`src/main/resources/META-INF/spring.factories`：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration
```

Spring Boot 2.7 / 3.x，`src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`：

```text
com.example.MyAutoConfiguration
```

## 测试

用 `ApplicationContextRunner` 做自动配置测试，不用起整个应用：

```java
class MyAutoConfigurationTest {

    private final ApplicationContextRunner contextRunner =
            new ApplicationContextRunner().withConfiguration(AutoConfigurations.of(MyAutoConfiguration.class));

    @Test
    void backsOffWhenBeanPresent() {
        contextRunner.withBean(MyClient.class, () -> new MyClient("x", 1))
                .run(context -> assertThat(context).hasSingleBean(MyClient.class));
    }
}
```

## 完整目录结构

```text
my-spring-boot-starter/
├── pom.xml
└── src/main/
    ├── java/com/example/
    │   ├── MyAutoConfiguration.java
    │   ├── MyProperties.java
    │   └── MyClient.java
    └── resources/
        └── META-INF/
            └── spring/
                └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

# 常见坑

1. **自动配置类被 `@ComponentScan` 扫到**：会绕过 `@ConditionalOn...`，被当普通配置类加载；放在独立包，或用 `@AutoConfiguration`。
2. **`@ConditionalOnMissingBean` 顺序不对**：它只能看到已经注册的 Bean；依赖其他自动配置时用 `@AutoConfigureAfter`。
3. **清单文件写错**：2.7+ 仍写 `spring.factories` 导致不生效；或类名拼错、少写换行。
4. **把自动配置类写成普通 `@Configuration`**：2.7+ 推荐 `@AutoConfiguration`，它自带 `proxyBeanMethods=false` 和排序支持。
5. **`@ConfigurationProperties` 没加 `@EnableConfigurationProperties`**：属性不会绑定，值全是默认值。
6. **starter 里写死版本**：依赖版本交给 BOM 管理，别在 starter 里锁死。
7. **自动配置覆盖用户 Bean**：所有默认 Bean 都应该加 `@ConditionalOnMissingBean`。
8. **`@AutoConfigureBefore/After` 用在普通配置类上**：不生效，它只对自动配置类排序。
9. **忘了排除冲突的自动配置**：比如自己配了 `DataSource`，要 `exclude = DataSourceAutoConfiguration.class`。

# 小结

我一向不太喜欢写总结，因为该说的正文里都说了。不过考虑到很多读者是冲着面试来的，还是列成清单：

1. **自动装配的原理？** `@SpringBootApplication` → `@EnableAutoConfiguration` → `@Import(AutoConfigurationImportSelector)` → 读取清单 → 条件过滤 → 排序 → 注册 BeanDefinition。
2. **自动配置类从哪来？** 2.x 是 `META-INF/spring.factories`，2.7+/3.x 是 `AutoConfiguration.imports`，从 classpath 下所有 jar 里读。
3. **为什么用户 Bean 能覆盖自动配置？** `AutoConfigurationImportSelector` 是 `DeferredImportSelector`，用户配置先解析，自动配置后执行，`@ConditionalOnMissingBean` 才能看到用户 Bean。
4. **加载顺序由谁决定？** `AutoConfigurationSorter`，按 `@AutoConfigureOrder` → `@AutoConfigureBefore/After` → 字母序。
5. **写 starter 要注意什么？** 命名规范、autoconfigure/starter 拆分、`@ConditionalOnClass` + `@ConditionalOnMissingBean`、`@ConfigurationProperties`、清单文件版本、`ApplicationContextRunner` 测试。
6. **自动配置类为什么不能被扫描到？** 会被当普通配置类加载，绕过条件判断。
7. **怎么排除自动配置？** `@SpringBootApplication(exclude=...)` 或 `spring.autoconfigure.exclude`。

# 参考

- [Spring Boot 官方文档：Auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
- [Spring Boot 官方文档：Creating Your Own Starter](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html)
- Spring Boot 源码：`AutoConfigurationImportSelector`、`AutoConfigurationSorter`、`ConfigurationClassPostProcessor`
- [../spring/Spring生命周期.md](../spring/Spring生命周期.md)、[router.md](router.md)（动态数据源实战）
