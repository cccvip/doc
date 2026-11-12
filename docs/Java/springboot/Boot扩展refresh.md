上一篇讲的是 Spring 通用的 refresh。这一篇只讲一件事：Boot 是怎么让内嵌的 Web 服务器在启动过程中"该建的时候建、该起的时候起"的。

先划个范围。这套东西只对 Servlet 应用有意义。批处理、命令行这类应用（`webApplicationType` 为 `NONE`）根本没有 Web 服务器，行为跟纯 Spring 一模一样；响应式应用（`REACTIVE`）另有一套实现。所以真正会碰到这段逻辑的，就是你引了 `spring-boot-starter-web` 的那种普通服务。

## 这一章其实只解决一个问题

refresh 完全是 Spring 的，它按自己那套流程走：解析注解、创建 Bean、收尾。问题是，Spring 框架本身并不知道你要不要对外提供 HTTP 服务，它不认识"Web 服务器"这种东西。

所以 Boot 得自己想清楚两件事：服务器什么时候建出来，什么时候让它开始接请求。这一章就围绕这两件事，没有更多。

## 先认识一个东西：Lifecycle

要看懂接下来的安排，得先知道 Spring 里有个叫 Lifecycle 的机制。

有一类对象需要等容器启动完成才开始工作、容器关闭时停下来，比如线程池、消息消费者、Web 服务器。Spring 给它们约定了统一的开关：实现 Lifecycle 接口，容器在启动收尾时会挨个把它们启动，关闭时挨个停掉。

这条消息对 Boot 很关键。它意味着 Boot 不必去改造 Spring 的 refresh，只要把 Web 服务器包装成一个 Lifecycle 交给容器，Spring 到点自然会把它启动。

## 于是 Boot 做了两件事

**建服务器**：在 refresh 的 `onRefresh` 这一步（第 9 步）把服务器建出来。这一步的时机是——Bean 马上要开始创建，但还没开始。放在这里，是因为有些 Bean 要用到 `ServletContext`（Servlet 规范里的上下文，Filter、Servlet 都注册在它上面），服务器得先建出来，它们才注入得进。

建的时候顺手做两件事：把容器里收集到的 Filter、Servlet、Listener 注册到 `ServletContext` 上——你写的 Filter 能被识别，就是这一步挂上去的；再额外注册一个负责启停的 Lifecycle 组件。

**起服务器**：到了 refresh 收尾（第 12 步），Spring 照例启动所有 Lifecycle，Boot 那个负责启停的组件趁机把服务器拉起来，绑定端口、开始接收请求。

于是"建"和"起"被拆到了两个时间点：建在第 9 步，起在第 12 步。

（顺带说一句版本差异：Spring Boot 2.2 及更早是在收尾时直接启动服务器，2.5 换成了上面这种 Lifecycle 的方式，顺便支持了优雅停机。）

## 为什么非要拆成两步

假如建完立刻起，会立刻撞上一个矛盾：此刻 Bean 还没创建完。有些 Bean 初始化时要拿到 `ServletContext`，服务器必须先存在；可服务器一旦提前接收请求，又会遇到"Bean 还没准备好，流量就来了"的尴尬。

拆开就都解决了——先建，满足注入需求；最后起，等 Bean 全部就绪。

## 顺带一提：选哪个服务器

第 9 步建服务器之前，Boot 得先决定用哪个实现：Tomcat、Jetty 还是 Undertow。它的办法是去容器里找"Web 服务器工厂"，并且要求**有且仅有一个**。

引了 `spring-boot-starter-web` 之后，自动配置会注册一个 Tomcat 的实现。要是同时把 Jetty、Undertow 的依赖也引进来，这里就会因为找到多个而报错，服务直接起不来。"多个 Web 容器依赖不能共存"说的就是这件事。

## 服务器起来时，事件走到哪了

前面说过，Lifecycle 是在收尾阶段被启动的。把它放回整条时间线：

| 时刻 | 服务器起来了吗 |
|---|---|
| Bean 初始化回调（`@PostConstruct`） | 没有，只是建好了 |
| 启动 Lifecycle，服务器在这时绑定端口 | 刚起来 |
| 发出 `ContextRefreshedEvent` | 已经起来了 |
| 发出 `ApplicationReadyEvent` | 已经起来了 |

后两个事件由 SpringApplication 在 refresh 之外发出，这里一并列出，方便对照。

这张表正好能解释两个面试常被问到的现象：

- 在 `@PostConstruct` 里访问本机端口会失败，因为服务器还没启动；
- 在 `ContextRefreshedEvent` 里访问没问题，服务器已经起来了。

如果要等"完全可用"再动手，用 `ApplicationRunner` 或监听 `ApplicationReadyEvent` 最稳。

## 面试速记

- 这套 Web 扩展只对 Servlet 应用生效，`NONE` 应用与纯 Spring 一致。
- 服务器**建**在 refresh 的 `onRefresh`（第 9 步），**起**在收尾阶段（第 12 步），靠的是一个 Lifecycle 组件。
- 拆成两步的原因：建早了是为了让 Bean 能拿到 `ServletContext`，起晚了是为了等 Bean 就绪。
- 服务器启动早于 `ContextRefreshedEvent`；应用完全可用看 `ApplicationReadyEvent`。
- 多个 Web 容器依赖不能共存。

## 参考

- Spring Boot 2.5.x 源码：`ServletWebServerApplicationContext`、`WebServerStartStopLifecycle`
- Spring Framework 5.3.x 源码：`AbstractApplicationContext.finishRefresh`
- [refresh流程.md](refresh流程.md)：Spring 通用流程
- [启动流程.md](启动流程.md)：Boot 的注解与 SpringApplication 编排
