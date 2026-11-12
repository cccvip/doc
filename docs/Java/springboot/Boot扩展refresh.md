上一篇 [refresh流程.md](refresh流程.md) 讲的是 Spring Framework 的通用流程。这一篇只讲 **Spring Boot 对它的扩展**：Boot 的 Servlet 容器子类，把内嵌 Web 服务器接进了 `refresh()`。

先明确一点：这些扩展**只对 Servlet 应用（`webApplicationType = SERVLET`）生效**。普通应用（`NONE`）和响应式应用（`REACTIVE`）没有内嵌 Servlet Web 服务器，走的是另外的实现，或者压根不做这件事。

> 基于 Spring Boot 2.5.x。

# Boot 换掉了容器类型

`createApplicationContext` 按 `webApplicationType` 选容器：

| 类型 | 容器类 | 内嵌 Web 服务器 |
|---|---|---|
| `SERVLET` | `AnnotationConfigServletWebServerApplicationContext`（继承 `ServletWebServerApplicationContext`） | Tomcat / Jetty / Undertow |
| `REACTIVE` | `AnnotationConfigReactiveWebServerApplicationContext` | Netty 等，由 `ReactiveWebServerFactory` 创建 |
| `NONE` | `AnnotationConfigApplicationContext`（无 Web 扩展） | 无 |

本篇以 Servlet 为例。

# Boot 覆盖了 refresh 的哪几个点

`ServletWebServerApplicationContext` 覆盖了父类的这些方法：

| 方法 | 在 refresh 里的位置 | Boot 做了什么 |
|---|---|---|
| `postProcessBeanFactory` | 第 4 步 | 注册 Web 作用域、处理 `ServletContextAware` |
| `onRefresh` | 第 9 步 | **创建** Web 服务器（不启动） |
| `refresh`（`final`） | 包在整个 refresh 外层 | 启动失败时停止 WebServer |
| `doClose` | 容器关闭时 | 发布 `ReadinessState.REFUSING_TRAFFIC` |

**注意：Boot 2.5 并没有覆盖 `finishRefresh`。** 它没有 `startWebServer()` 这种做法。Web 服务器的**启动**是由一个注册进容器的 `SmartLifecycle` 完成的（见下）。

# 一、postProcessBeanFactory：注册 Web 作用域

```java
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    beanFactory.addBeanPostProcessor(new WebApplicationContextServletContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    registerWebApplicationScopes();
}
```

- `WebApplicationContextServletContextAwareProcessor`：处理 `ServletContextAware`，让 Bean 能注入 `ServletContext` / `ServletConfig`；
- `ignoreDependencyInterface(ServletContextAware.class)`：这些接口由处理器回调，不要按类型自动注入；
- `registerWebApplicationScopes()`：注册 `request` / `session` 作用域（`application` 作用域稍后在 `selfInitialize` 里注册）。

**为什么必须是这一步？** 它在第 4 步，早于第 5 步的 BeanDefinition 解析。作用域要先注册好，后面声明成 `@Scope("request")` 的 BeanDefinition 才有效。

# 二、onRefresh：创建 Web 服务器

```java
@Override
protected void onRefresh() {
    super.onRefresh();
    createWebServer();
}

private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = getServletContext();

    if (webServer == null && servletContext == null) {
        ServletWebServerFactory factory = getWebServerFactory();
        this.webServer = factory.getWebServer(getSelfInitializer());
        getBeanFactory().registerSingleton("webServerGracefulShutdown",
                new WebServerGracefulShutdownLifecycle(this.webServer));
        getBeanFactory().registerSingleton("webServerStartStop",
                new WebServerStartStopLifecycle(this, this.webServer));
    }
    else if (servletContext != null) {
        getSelfInitializer().onStartup(servletContext);
    }
    initPropertySources();
}
```

分几块看。

## 1）找到唯一的 ServletWebServerFactory

```java
String[] beanNames = getBeanFactory().getBeanNamesForType(ServletWebServerFactory.class);
if (beanNames.length == 0) {
    throw new ApplicationContextException("... missing ServletWebServerFactory bean.");
}
if (beanNames.length > 1) {
    throw new ApplicationContextException("... multiple ServletWebServerFactory beans : ...");
}
```

引入 `spring-boot-starter-web` 后，自动配置会注册一个 `TomcatServletWebServerFactory`（换成 Jetty / Undertow 则是别的实现）。这里要求**有且仅有一个**——所以同时引入多个 Web 容器会直接启动失败。

## 2）factory.getWebServer：创建服务器 + ServletContext

这一步创建出 `WebServer` 对象，同时创建 `ServletContext` 并回调 `selfInitialize`：

```java
private void selfInitialize(ServletContext servletContext) throws ServletException {
    prepareWebApplicationContext(servletContext);
    registerApplicationScope(servletContext);            // application 作用域
    WebApplicationContextUtils.registerEnvironmentBeans(getBeanFactory(), servletContext);
    for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
        beans.onStartup(servletContext);                  // 注册 Filter / Servlet / Listener
    }
}
```

`getServletContextInitializerBeans()` 会收集容器里的 `ServletContextInitializer`、`Servlet`、`Filter` 以及部分 `EventListener` Bean，逐个 `onStartup` 注册到 `ServletContext`。**你写的 Filter / Servlet 能被 Web 服务器识别，就是在这里挂上去的。**

注意：**此时服务器只是创建了，还没有 `start()`**——端口没绑定，请求进不来。

## 3）注册两个 SmartLifecycle

```java
new WebServerGracefulShutdownLifecycle(this.webServer)  // 优雅停机
new WebServerStartStopLifecycle(this, this.webServer)   // 启动 / 停止
```

它们是理解"服务器何时启动"的关键。

# 三、Web 服务器什么时候真正启动

**创建在 `onRefresh`（第 9 步），启动在第 12 步 `finishRefresh` 里的 `lifecycleProcessor.onRefresh()`。**

`WebServerStartStopLifecycle` 是一个 `SmartLifecycle`：

```java
class WebServerStartStopLifecycle implements SmartLifecycle {

    @Override
    public void start() {
        this.webServer.start();                 // ← 真正绑定端口、开始接收请求
        this.running = true;
        this.applicationContext.publishEvent(
                new ServletWebServerInitializedEvent(this.webServer, this.applicationContext));
    }

    @Override
    public int getPhase() {
        return Integer.MAX_VALUE - 1;           // 排在多数 Lifecycle 之后启动
    }
}
```

而 `finishRefresh` 的父类实现是：

```java
protected void finishRefresh() {
    clearResourceCaches();
    initLifecycleProcessor();
    getLifecycleProcessor().onRefresh();   // ← 启动 WebServer，并发 ServletWebServerInitializedEvent
    publishEvent(new ContextRefreshedEvent(this));
}
```

所以顺序是：

1. `lifecycleProcessor.onRefresh()` → `WebServerStartStopLifecycle.start()` → `webServer.start()` → 发布 `ServletWebServerInitializedEvent`；
2. 然后才发布 `ContextRefreshedEvent`。

**结论：Web 服务器启动早于 `ContextRefreshedEvent`。** `getPhase() = Integer.MAX_VALUE - 1` 只是让它排在多数 `Lifecycle` Bean 之后启动，但仍在 `ContextRefreshedEvent` 之前。

> 版本差异：Spring Boot 2.2 及更早，是在 `ServletWebServerApplicationContext.finishRefresh()` 里直接调用 `startWebServer()`；2.5 改成了上面这种 `SmartLifecycle` 方式（顺带支持优雅停机）。

# 事件顺序与"服务是否可用"

| 事件 / 回调 | 触发位置 | WebServer 是否已启动 |
|---|---|---|
| `@PostConstruct` | `finishBeanFactoryInitialization` | 否 |
| `ServletWebServerInitializedEvent` | `finishRefresh` → `lifecycleProcessor.onRefresh()` | 是（刚启动） |
| `ContextRefreshedEvent` | `finishRefresh` | 是 |
| `ApplicationStartedEvent` | `refresh()` 返回后、Runner 前 | 是 |
| `ApplicationReadyEvent` | Runner 执行完后 | 是 |

（后两个是 Boot 在 `SpringApplication.run` 里发布的，不属于 `refresh` 本身。）

这解释了两个常见现象：

- 在 `@PostConstruct` 里访问本机端口**会失败**——此时服务器只创建、没启动，而 `@PostConstruct` 发生在 `finishBeanFactoryInitialization`，早于 `finishRefresh`；
- 在 `ContextRefreshedEvent` 里访问本机端口**是可行的**——服务器已经启动。

想在"服务完全可用"后做事，用 `ApplicationRunner` 或监听 `ApplicationReadyEvent` 最稳妥。

# 只对 Servlet 应用生效

| `webApplicationType` | 容器 | 内嵌 Web 服务器 |
|---|---|---|
| `SERVLET` | `AnnotationConfigServletWebServerApplicationContext` | Tomcat / Jetty / Undertow |
| `REACTIVE` | `AnnotationConfigReactiveWebServerApplicationContext` | Netty 等 |
| `NONE` | `AnnotationConfigApplicationContext` | 无 |

`NONE` 类型（纯批处理、命令行应用）不会创建任何 Web 服务器，`onRefresh` 就是默认空实现，行为与 [refresh流程.md](refresh流程.md) 描述的完全一致。

# 常见误解

1. **Boot 覆盖了 `finishRefresh` 来启动服务器**：2.5 不是，改用 `SmartLifecycle`；2.2 及更早才是。
2. **`ContextRefreshedEvent` 时服务器还没启动**：错，服务器在此之前已由 `lifecycleProcessor.onRefresh()` 启动。
3. **`@PostConstruct` 里能调本机接口**：不能，服务器还没 `start()`。
4. **`onRefresh` 会启动服务器**：只创建，不启动。
5. **Web 服务器在 Bean 实例化之后才创建**：不对，创建在 `onRefresh`（早于 Bean 实例化）。
6. **多个 Web 容器依赖可以共存**：不可以，`getWebServerFactory` 要求有且仅有一个 `ServletWebServerFactory`。

# 小结

1. **Boot 对 refresh 的扩展只对 Servlet 应用生效**，`NONE` 应用与纯 Spring 一致。
2. **覆盖点**是 `postProcessBeanFactory`（注册 Web 作用域）和 `onRefresh`（创建 WebServer），不是 `finishRefresh`。
3. **创建**在 `onRefresh`（第 9 步），**启动**在 `finishRefresh` 的 `lifecycleProcessor.onRefresh()`，由 `WebServerStartStopLifecycle` 完成。
4. **顺序**：`@PostConstruct` → WebServer 启动 / `ServletWebServerInitializedEvent` → `ContextRefreshedEvent` → `ApplicationStartedEvent` → Runner → `ApplicationReadyEvent`。
5. **`@PostConstruct` 不能访问本机端口，`ContextRefreshedEvent` 可以。**

# 参考

- Spring Boot 2.5.x 源码：`ServletWebServerApplicationContext`、`WebServerStartStopLifecycle`、`WebServerGracefulShutdownLifecycle`
- Spring Framework 5.3.x 源码：`AbstractApplicationContext.finishRefresh`
- [refresh流程.md](refresh流程.md)（Spring 通用流程）、[启动流程.md](启动流程.md)（Boot 的注解与编排）
