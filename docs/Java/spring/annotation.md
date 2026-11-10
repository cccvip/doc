> 这篇从"注解"的角度看 SpringMVC：`@RestController`、`@Controller`、`@ResponseBody`、`@RequestMapping` 这些注解长什么样、怎么被解析生效。**请求进来之后的完整流程**（`doDispatch`、`HandlerMapping`、参数解析、响应写回）在 [springMvc/Mvc.md](../springMvc/Mvc.md) 里已经逐行讲过，本文不重复。

看注解不能只看用法，要能往下追到实现。这篇的目标就是：给一个注解，你能说清它是怎么生效的。

```java
import javax.annotation.Resource;

@RestController
@RequestMapping(value = "app")
public class AppController {

    @Resource
    AppService appService;

    @RequestMapping(value = "/ping")
    public String ping() {
        return "pong";
    }
}
```

# @RestController

`@RestController` 是 `@Controller` + `@ResponseBody` 的组合注解：类上标了它，就相当于所有方法都加了 `@ResponseBody`，不用每个方法再写一遍。

```java
@Controller
@ResponseBody
public @interface RestController {
    @AliasFor(annotation = Controller.class)   // value 转发给 @Controller 的 value
    String value() default "";
}

@Component
public @interface Controller {
    @AliasFor(annotation = Component.class)    // value 转发给 @Component
    String value() default "";
}
```

两个细节：

- `@ResponseBody`：标识返回值以 RESTful 风格（JSON）写回响应体；
- `@Controller`：它自身带 `@Component`，所以被扫描成 Spring 管理的 Bean；
- **`@AliasFor`**：把本注解的属性和被组合注解的属性关联起来，这就是"组合注解"的实现基础。

# @RequestMapping 是怎么生效的

要理解 `@RequestMapping`，得理解 SpringMVC 怎么**注册**和**匹配**映射。

## 注册：RequestMappingHandlerMapping

`@RequestMapping` 方法的注册发生在 `RequestMappingHandlerMapping` 初始化时：

```java
public class RequestMappingHandlerMapping extends RequestMappingInfoHandlerMapping
        implements MatchableHandlerMapping, EmbeddedValueResolverAware {

    @Override
    public void afterPropertiesSet() {
        super.afterPropertiesSet();     // 触发下面的扫描
    }
}

public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {

    private final Map<T, MappingRegistration<T>> registry = new HashMap<>();

    @Override
    public void afterPropertiesSet() {
        initHandlerMethods();           // 扫描所有 Handler 的方法
    }

    protected void initHandlerMethods() {
        for (String beanName : getCandidateBeanNames()) {
            if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
                processCandidateBean(beanName);
            }
        }
    }

    protected void processCandidateBean(String beanName) {
        // 判断这个 Bean 是不是带 @Controller / @RequestMapping
        if (beanType != null && isHandler(beanType)) {
            detectHandlerMethods(beanName);      // 解析它的方法并注册
        }
    }

    protected boolean isHandler(Class<?> beanType) {
        return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
                AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
    }

    // 最终每个 (映射信息 → 方法) 都存进 registry
    protected void registerHandlerMethod(Object handler, Method method, T mapping) {
        this.mappingRegistry.register(mapping, handler, method);
    }
}
```

一句话：**启动时扫描 → 判断哪些 Bean 是 Handler → 把它的 `@RequestMapping` 方法注册进映射表**。

## 匹配与执行

请求进来后，怎么从一个 URL 找到这个方法、又怎么把响应写出去——完整链条（`initStrategies`、`doDispatch`、`lookupHandlerMethod`、参数解析、`@ResponseBody` 写回）都在 [springMvc/Mvc.md](../springMvc/Mvc.md)。

那里面的 `RequestMappingHandlerMapping` / `RequestMappingHandlerAdapter` 就是这里注册的映射在运行时的消费方。

# 每日一题

下面这种写法会 **404**，想想为什么？

```java
@RestController
public abstract class AppController {
}
```

提示：Spring 管理的"Bean"必须是一个**可实例化的 class**。如果是 `abstract class` 或 `interface`，它**不会被扫描为 Bean**——没有 Bean，自然没有注册映射，请求也就匹配不到。

> 再想一层：回想一下三级缓存——Spring 要的是能被实例化、能被代理的**具体类**。

# 小结

1. **`@RestController` 是什么？** `@Controller` + `@ResponseBody` 的组合注解。
2. **`@Controller` 为什么会被扫描？** 它自身带 `@Component`。
3. **`@AliasFor` 干什么？** 把组合注解的属性转发给被组合的注解（如 `value` → `@Controller.value`）。
4. **`@RequestMapping` 什么时候注册的？** `RequestMappingHandlerMapping` 初始化时，`afterPropertiesSet → initHandlerMethods` 扫描并注册进 `registry`。
5. **请求进来后怎么执行？** 见 [springMvc/Mvc.md](../springMvc/Mvc.md)。
6. **抽象类上的 `@RestController` 为什么 404？** 抽象类/接口不会被扫描成 Bean，也就没有映射注册。

# 参考

- Spring 5.x 源码：`RestController`、`AbstractHandlerMethodMapping`
- [springMvc/Mvc.md](../springMvc/Mvc.md)（请求分发全流程）
