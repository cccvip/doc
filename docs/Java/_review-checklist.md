# Java 文档审阅修改清单

> 状态标记：⬜ 待改 / 🟨 进行中 / ✅ 已改
> 原则：直接订正原文；有争议或纯个人观点的保留并加 `⚠️` 批注；每批改动对应一个 commit。

## 第 1 批 · P0 事实硬伤

| 文件 | 问题 | 状态 |
|---|---|---|
| 基本/HashMap.md | HashMap/Hashtable 的 null 规则写反 | ✅ |
| 基本/HashMap.md | Hashtable 迭代器"fail-safe"表述错误 | ✅ |
| 基本/HashMap.md | 扰动函数验证代码用 `&`，应为 `^`（并重算输出表） | ✅ |
| 基本/HashMap.md | 扩容规律 `oldCap&newCap` 应为 `hash & oldCap` | ✅ |
| 基本/threadLocal.md | "thread 作为 key"错误，应为 ThreadLocal 实例 | ✅ |
| 基本/threadLocal.md | 内存泄漏场景写反（线程结束 vs 线程长期存活） | ✅ |
| 基本/threadLocal.md | "线程安全的单例"概念自相矛盾 | ✅ |
| 基本/Synchorized.md | new 对象步骤错误（应为 分配→零值→对象头→构造） | ✅ |
| 基本/Synchorized.md | volatile 屏障类型错误（四个 StoreStore） | ✅ |
| 基本/Synchorized.md | 锁标志位错误（000/10/11） | ✅ |
| 基本/Synchorized.md | 偏向锁"计数+1 即可重入"错误 | ✅ |
| 基本/Synchorized.md | DCL 示例 `getInstance` 漏 static；字段名 instance/singleton 不一致 | ✅ |
| 基本/JVM.md | "对象>年轻代1/2 进老年代"混淆 PretenureSizeThreshold / 动态年龄 | ✅ |
| 基本/JVM.md | G1 "固定 8:2 分区"错误 | ✅ |
| 基本/JVM.md | 老年代算法注 "(CMS)" 错误 | ✅ |
| 基本/JVM.md | "static final 静态变量入堆"表述不准；"字符串串"笔误 | ✅ |

## 第 2 批 · P0 代码跑不通

| 文件 | 问题 | 状态 |
|---|---|---|
| 基本/注解.md | 只定义 value，却使用 number() | ✅ |
| 基本/注解.md | 缺 @Retention(RUNTIME)、@Target，运行时 getAnnotation 取不到 | ✅ |
| 基本/反射.md | `Class.forName("com.example.MyClass")` 包名不匹配、`newInstance()` 已废弃 | ✅ |
| spring/SpringIOC.md | `UserDao userService` / `userDao` 变量名不一致，编译错误 | ✅ |
| spring/SpringIOC.md | 把"Bean 实例化"与"JDK/CGLIB 代理"混为一谈 | ✅ |
| spring/SpringIOC.md | BeanPostProcessor 描述"实例化"应为"初始化" | ✅ |
| spring/SpringIOC.md | `parseBeanDefinitions` 误写成 `processBeanDefinition`，`root` 未定义 | ✅ |
| springboot/router.md | `@Before` 配 ProceedingJoinPoint 且 return，无法编译 | ✅ |
| springboot/router.md | 切面用 DatabaseSharding、拦截器用 DS，注解名不统一 | ✅ |

## 第 3 批 · P1 自相矛盾 / 概念混淆

| 文件 | 问题 | 状态 |
|---|---|---|
| spring/Spring循环依赖.md | 一级缓存示例已跑通，却断言"无法解决" | ✅ |
| spring/Spring循环依赖.md | "两级缓存破坏生命周期"语义不通 | ✅ |
| spring/Spring循环依赖.md | 三级缓存"移除二级、加入一级"描述混乱 | ✅ |
| spring/基本概念.md | BeanPostProcessor 注释/正文"实例化"应为"初始化" | ✅ |
| spring/基本概念.md | BeanFactoryPostProcessor 注释"初始化后"错误 | ✅ |
| spring/总结.md | AOP 代理生成时机错误（BeforeInstantiation） | ✅ |
| spring/SpringAop代理.md | postProcessBeforeInstantiation 职责被误写成解析 @Aspect | ✅ |
| springboot/starter.md | 流程顺序写成 初始化→属性填充→实例化 | ✅ |
| springMvc/Mvc.md | HandlerMapping "实现类" 写成抽象类/接口 | ✅ |
| springMvc/Mvc.md | "HandlerAdapter 不准确，应该是 Executor" 表述错误 | ✅ |
| springMvc/Mvc.md | 查找 Handler 示例给的是 nameLookup，应为 mappingLookup | ✅ |
| mybatis/Proxy.md | MyBatis Mapper 支持 CGLib 代理（错误） | ✅ |

## 第 4 批 · P1 Netty

| 文件 | 问题 | 状态 |
|---|---|---|
| netty/firstPage.md | Netty 默认触发模式写成水平触发（应为 ET） | ✅ |
| netty/future.md | "FutureTask 锁竞争"整节论证错误（实为无锁 CAS） | ✅ |
| netty/实战.md | 误粘贴 Go ServeMux 路由文档 | ✅ |
| netty/readme.md | nginx 配置注释用 `//`（应为 `#`） | ✅ |
| netty/readme.md | nginx 配置行首混入全角空格，导致配置无效 | ✅ |

## 第 5 批 · P2 精度 / 表述

| 文件 | 问题 | 状态 |
|---|---|---|
| 基本/JedisSourceCode.md | MaxIdle 释义错误 | ✅ |
| 基本/JedisSourceCode.md | 连接数公式量纲错误 | ✅ |
| 基本/GC.md | jmap -dump 触发 Full GC 需 :live | ✅ |
| 基本/内存调优第二篇.md | "介入指标"列的是健康值，逻辑反了 | ✅ |
| 基本/内存调优.md | "younggc 很多"与"10 分钟一次"矛盾 | ✅ |
| 基本/package.md | classpathPrefix `libs/` 与示例 `lib/` 不一致 | ✅ |
