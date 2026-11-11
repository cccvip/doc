`@RestController`、`@RequestMapping` 天天在用，但很多人只停留在"标在类上就能接收请求"这一层。再问一句"注解是怎么变成映射表的、类上的路径和方法上的路径怎么拼起来、为什么抽象类标了 `@RestController` 会 404"，就说不清了。

这篇从注解的定义和 `RequestMappingHandlerMapping` 的注册过程出发，把"注解怎么生效"走一遍。请求进来之后的 `doDispatch` 流程在 [springMvc/Mvc.md](../springMvc/Mvc.md) 已经讲过，这里不重复。如果只为面试，可以直接滑到最后的**小结**。

# 总览

**一句话：注解本身不会生效，是 Spring 启动时扫描注解、把它解析成 BeanDefinition 或请求映射，运行时再由对应的组件消费。**

```text
启动阶段：
@ComponentScan
  → 扫描 @Component 及其派生注解 → BeanDefinition
  → RequestMappingHandlerMapping.afterPropertiesSet
      → initHandlerMethods → detectHandlerMethods
      → getMappingForMethod（类级 + 方法级合并）
      → mappingRegistry.register(URL → HandlerMethod)

运行阶段：
DispatcherServlet.doDispatch
  → getHandler → RequestMappingHandlerMapping.lookupHandlerMethod
  → HandlerAdapter → 反射调用
```

# 核心概念

## 注解生效的两个前提

1. **运行时可见**：注解上必须有 `@Retention(RetentionPolicy.RUNTIME)`。默认是 `CLASS`，编译进字节码但反射读不到；
2. **有人解析它**：`@Component` 由 `ClassPathBeanDefinitionScanner` 解析成 BeanDefinition；`@RequestMapping` 由 `RequestMappingHandlerMapping` 解析成映射。

少了任意一条，注解都只是"注释"。

## 组合注解与 @AliasFor

`@RestController` 是 `@Controller` + `@ResponseBody` 的组合：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {
    @AliasFor(annotation = Controller.class)   // value 转发给 @Controller 的 value
    String value() default "";
}
```

`@Controller` 自身又带 `@Component`，所以会被扫描成 Bean。`@AliasFor` 的作用是把组合注解的属性转发给被组合的注解，这是所有"组合注解"的基础。

## @RequestMapping 的派生注解

`@GetMapping` / `@PostMapping` / `@PutMapping` / `@DeleteMapping` / `@PatchMapping` 本质上都是：

```java
@RequestMapping(method = RequestMethod.GET)
```

它们通过元注解继承 `@RequestMapping`，`AnnotatedElementUtils.findMergedAnnotation` 会把它们当作 `@RequestMapping` 来解析。

# 源码分析

## @RestController 的定义

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {
    @AliasFor(annotation = Controller.class)
    String value() default "";
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Controller {
    @AliasFor(annotation = Component.class)
    String value() default "";
}
```

两个细节：

- `@ResponseBody`：标识返回值以 RESTful 风格（JSON）写回响应体；
- `@Controller`：自身带 `@Component`，所以被扫描成 Spring 管理的 Bean。

## 注册：RequestMappingHandlerMapping.afterPropertiesSet

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
}
```

一句话：**启动时扫描 → 判断哪些 Bean 是 Handler → 把它的 `@RequestMapping` 方法注册进映射表**。

## detectHandlerMethods / getMappingForMethod

真正解析方法映射的是 `detectHandlerMethods`：

```java
protected void detectHandlerMethods(Object handler) {
    Class<?> handlerType = (handler instanceof String ?
            obtainApplicationContext().getType((String) handler) : handler.getClass());

    if (handlerType != null) {
        Class<?> userType = ClassUtils.getUserClass(handlerType);

        // 遍历类里的每个方法，尝试解析出 RequestMappingInfo
        Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
                (MethodIntrospector.MetadataLookup<T>) method -> {
                    try {
                        return getMappingForMethod(method, userType);
                    } catch (Throwable ex) {
                        throw new IllegalStateException("Invalid mapping on handler class ["
                                + userType.getName() + "]: " + method, ex);
                    }
                });

        methods.forEach((method, mapping) -> {
            Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
            registerHandlerMethod(handler, invocableMethod, mapping);   // 放进 mappingRegistry
        });
    }
}
```

类级路径和方法级路径的合并发生在 `getMappingForMethod`：

```java
@Override
@Nullable
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
    RequestMappingInfo info = createRequestMappingInfo(method);
    if (info != null) {
        RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
        if (typeInfo != null) {
            info = typeInfo.combine(info);      // 类级 + 方法级拼接
        }
        String prefix = getPathPrefix(handlerType);
        if (prefix != null) {
            info = RequestMappingInfo.paths(prefix).options(this.config).build().combine(info);
        }
    }
    return info;
}

private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
    // findMergedAnnotation 会处理元注解：@GetMapping 也能当成 @RequestMapping
    RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
    RequestCondition<?> condition = (element instanceof Class ?
            getCustomTypeCondition((Class<?>) element) : getCustomMethodCondition((Method) element));
    return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
}
```

所以类上写 `@RequestMapping("/api/users")`、方法上写 `@GetMapping("/{id}")`，最终注册的路径是 `/api/users/{id}`。

## 运行时：lookupHandlerMethod

请求进来后，`DispatcherServlet` 会调用 `RequestMappingHandlerMapping.lookupHandlerMethod`，从 `mappingRegistry` 里按 URL 找到 `HandlerMethod`，再交给 `HandlerAdapter` 反射调用。

完整链条（`doDispatch`、参数解析、`@ResponseBody` 写回）见 [springMvc/Mvc.md](../springMvc/Mvc.md)，这里不重复。

# 实战与常见坑

## 场景：类级 + 方法级路径拼接

```java
@RestController
@RequestMapping("/api/users")
class UserController {

    @GetMapping("/{id}")
    public User get(@PathVariable Long id) {
        // ...
    }
}
```

注册过程对应源码：

1. `createRequestMappingInfo(handlerType)` 得到类级映射 `/api/users`；
2. `createRequestMappingInfo(method)` 得到方法级映射 `/{id}`；
3. `typeInfo.combine(info)` 合并成 `/api/users/{id}`；
4. 存进 `mappingRegistry`，运行时按这个路径匹配。

`@GetMapping` 只是 `@RequestMapping(method = GET)` 的派生注解，解析时会被 `findMergedAnnotation` 当成 `@RequestMapping`。

## 常见坑

1. **抽象类 / 接口上的 `@RestController` 不会生效**：Spring 管理的 Bean 必须是可实例化的具体类，没有 Bean 就没有映射，请求自然 404。
2. **`@Controller` 忘了 `@ResponseBody`**：返回的是视图名，不是 JSON；要么每个方法加 `@ResponseBody`，要么直接用 `@RestController`。
3. **类级和方法级路径拼接错误**：两边都写 `/` 或都不写，容易拼出 `//api` 或 `api`；保持统一风格。
4. **同一个 URL + HTTP 方法映射到两个方法**：启动时抛 `Ambiguous mapping`。
5. **`@RequestMapping` 不写 `method`**：会匹配所有 HTTP 方法，容易和具体方法冲突。
6. **`@PathVariable` 名字和占位符不一致**：编译时没开 `-parameters` 时尤其容易出错，建议显式写 `@PathVariable("id")`。
7. **`produces` / `consumes` 不匹配**：返回 406 / 415，别忽略内容协商。
8. **自定义注解忘了 `@Retention(RUNTIME)`**：运行时反射读不到，注解等于没写。

# 小结

我一向不太喜欢写总结，因为该说的正文里都说了。不过考虑到很多读者是冲着面试来的，还是列成清单：

1. **`@RestController` 是什么？** `@Controller` + `@ResponseBody` 的组合注解。
2. **`@Controller` 为什么会被扫描？** 它自身带 `@Component`。
3. **`@AliasFor` 干什么？** 把组合注解的属性转发给被组合的注解（如 `value` → `@Controller.value`）。
4. **`@RequestMapping` 什么时候注册的？** `RequestMappingHandlerMapping` 初始化时，`afterPropertiesSet → initHandlerMethods → detectHandlerMethods`。
5. **类级和方法级路径怎么合并？** `getMappingForMethod` 里 `typeInfo.combine(info)`。
6. **`@GetMapping` 和 `@RequestMapping` 什么关系？** `@GetMapping` 是 `@RequestMapping(method = GET)` 的派生注解，靠 `findMergedAnnotation` 解析。
7. **抽象类上的 `@RestController` 为什么 404？** 抽象类 / 接口不会被扫描成 Bean，也就没有映射注册。

# 参考

- [Spring Framework 官方文档：RequestMapping](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-requestmapping.html)
- Spring 5.x 源码：`RestController`、`AbstractHandlerMethodMapping`、`RequestMappingHandlerMapping`
- [springMvc/Mvc.md](../springMvc/Mvc.md)（请求分发全流程）
