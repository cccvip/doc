前后端分离的项目里，SpringMVC 我们真正关心的是一条请求进来之后：怎么找到 Controller？参数怎么绑上去？拦截器怎么串？响应又是怎么出去的。视图渲染反而不重要——大多数接口一个 `@ResponseBody` 就把响应写回去了。

这篇不满足于"列一遍流程"，而是跟着源码把上面这几个问题走一遍：从 `DispatcherServlet.doDispatch` 出发，一直追到 `lookupHandlerMethod`、`invokeForRequest`、`processDispatchResult`。

> 本文对着 Spring 5.x（5.3）的源码看。只想冲刺面试的，可以直接翻到最后的**小结**。

# 总览：一次请求的骨架

![springMvc](springMvc.png)

一次请求走完，主要是四站：

1. **找 Handler**：`getHandler` → `HandlerMapping.getHandler` → `lookupHandlerMethod`，最终拿到包着 `HandlerMethod` 的 `HandlerExecutionChain`；
2. **找适配器**：`getHandlerAdapter` 按 Handler 类型挑一个 `HandlerAdapter`；
3. **执行**：`HandlerExecutionChain.applyPreHandle` 跑拦截器，然后 `HandlerAdapter.handle` → `RequestMappingHandlerAdapter.invokeHandlerMethod`，参数解析、调用 Controller、处理返回值都在这里；
4. **收尾**：`processDispatchResult` 渲染视图，或（`@ResponseBody`）响应已写回、直接走 `afterCompletion`。

下面按这四站逐段看源码。

# 请求入口：DispatcherServlet.doDispatch

一切从 `doDispatch` 开始（省略了异步、文件上传等分支）：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;
        try {
            processedRequest = checkMultipart(request);              // 上传请求会先被包装
            multipartRequestParsed = (processedRequest != request);

            // ① 找处理器：返回 处理器 + 拦截器链
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response);          // 没找到 → 404
                return;
            }

            // ② 找能执行这个处理器的适配器
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // ③ 拦截器前置处理，任一 preHandle 返回 false 就直接结束
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // ④ 真正执行：参数绑定、调用 Controller、处理返回值，都在这一句里面
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            applyDefaultViewName(processedRequest, mv);              // 没视图名就补默认视图名

            // ⑤ 拦截器后置处理（逆序）
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        } catch (Exception ex) {
            dispatchException = ex;
        }

        // ⑥ 统一收尾：渲染视图，或响应已写回则跳过；并触发 afterCompletion
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    } finally {
        if (multipartRequestParsed) {
            cleanupMultipart(processedRequest);
        }
    }
}
```

`doDispatch` 本身很薄，它只是一条调度主线。真正的东西全在被它调用的那几层里。**下面按 ①～⑥ 展开。**

# 第一站：URL 是怎么变成 HandlerMethod 的

## HandlerMapping 家族

`getHandler` 委托给 `HandlerMapping`。先澄清一个高频混淆点：

> `AbstractHandlerMapping` 是**抽象基类**、`MatchableHandlerMapping` 是**接口**，都不是可直接使用的"实现类"。常见实现是 **`RequestMappingHandlerMapping`**（处理 `@RequestMapping` 系注解）和 **`SimpleUrlHandlerMapping`**（按 URL 硬映射）。

`AbstractHandlerMapping.getHandler` 是入口（`final` 的，逻辑固定）：

```java
@Override
@Nullable
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    // 交给子类，真正按请求去找 Handler
    Object handler = getHandlerInternal(request);
    if (handler == null) {
        handler = getDefaultHandler();          // 兜底 Handler
    }
    if (handler == null) {
        return null;                            // 真的没有 → doDispatch 里 404
    }
    // Handler 可能是 Bean 名字符串，还原成 Bean 实例
    if (handler instanceof String) {
        handler = obtainApplicationContext().getBean((String) handler);
    }
    if (!ServletRequestPathUtils.hasCachedPath(request)) {
        initLookupPath(request);                // 缓存 lookupPath，拦截器/匹配都要用
    }
    // 把 Handler 和匹配到的拦截器一起，包成 HandlerExecutionChain
    return getHandlerExecutionChain(handler, request);
}
```

## RequestMappingHandlerMapping：把注解编成 RequestMappingInfo

`RequestMappingHandlerMapping` 是 `@RequestMapping` 那一套的实现。它做了两件事：**启动时扫描注册**、**运行时按请求匹配**。

先看怎么判断一个 Bean 算不算 Handler：

```java
// 类上有 @Controller 或 @RequestMapping，就当它是 Handler
@Override
protected boolean isHandler(Class<?> beanType) {
    return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
            AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
}
```

启动时，`initHandlerMethods` 会遍历容器里的所有 Bean，对每个 Handler 的每个方法调 `getMappingForMethod`，把方法上的 `@RequestMapping` / `@GetMapping` / `@PostMapping` 等**汇编成一个 `RequestMappingInfo` 对象**（它把 URL、method、params、headers、consumes、produces 等匹配条件都收进来了）：

```java
// 简化后的骨架：方法上的注解 → RequestMappingInfo
@Override
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
    RequestMappingInfo info = createRequestMappingInfo(method);
    if (info != null) {
        RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);   // 类上的前缀
        if (typeInfo != null) {
            info = typeInfo.combine(info);   // 类前缀 + 方法路径，合并
        }
        // ... 处理 produces/consumes 等条件
    }
    return info;
}
```

最后每个 `RequestMappingInfo → HandlerMethod` 都塞进 `MappingRegistry`，等请求来匹配。

## 核心：getHandlerInternal → lookupHandlerMethod

运行时，`AbstractHandlerMethodMapping.getHandlerInternal` 是真正的匹配入口：

```java
@Override
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
    String lookupPath = initLookupPath(request);          // 用于匹配的请求路径
    this.mappingRegistry.acquireReadLock();               // 读写锁：允许多个请求并发查
    try {
        HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
        // createWithResolvedBean：把 HandlerMethod 里的 Bean 名字解析成真实 Bean
        return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
    } finally {
        this.mappingRegistry.releaseReadLock();
    }
}
```

`lookupHandlerMethod` 是这段里最关键的方法，它把匹配分成"先快后全 + 排序选最优"：

```java
@Nullable
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
    List<Match> matches = new ArrayList<>();

    // 1. 先走"直接路径"匹配（pathLookup 里缓存的精确路径），命中率高，快
    List<T> directPathMatches = this.mappingRegistry.getMappingsByDirectPath(lookupPath);
    if (directPathMatches != null) {
        addMatchingMappings(directPathMatches, matches, request);
    }

    // 2. 直接匹配没命中，再拿所有注册的 Mapping 全量试一遍（含通配符、变量路径）
    if (matches.isEmpty()) {
        addMatchingMappings(this.mappingRegistry.getRegistrations().keySet(), matches, request);
    }

    if (!matches.isEmpty()) {
        // 3. 可能多个都匹配（比如 /user/{id} 和 /user/list），用比较器排序取最优
        Match bestMatch = matches.get(0);
        if (matches.size() > 1) {
            Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
            matches.sort(comparator);
            bestMatch = matches.get(0);
        }
        request.setAttribute(HandlerMapping.BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.handlerMethod);
        return bestMatch.handlerMethod;
    } else {
        // 一个都没匹配上，走抛错逻辑（最终 404）
        return handleNoMatch(this.mappingRegistry.getRegistrations().keySet(), lookupPath, request);
    }
}
```

其中"某个 Mapping 是否匹配"由 `addMatchingMappings` 逐个判断：

```java
private void addMatchingMappings(Collection<T> mappings, List<Match> matches, HttpServletRequest request) {
    for (T mapping : mappings) {
        // 用一个 RequestMappingInfo 去比对这次请求，匹配则返回自身，否则 null
        T match = getMatchingMapping(mapping, request);
        if (match != null) {
            matches.add(new Match(match, this.mappingRegistry.getRegistrations().get(mapping)));
        }
    }
}
```

## 匹配的细节：一次请求要过几道条件

`RequestMappingInfo` 把匹配条件拆成了好几个 `...Condition`，`getMatchingCondition` 会**逐项过滤，任意一项不通过就整个不匹配**：

```java
@Override
@Nullable
public RequestMappingInfo getMatchingCondition(HttpServletRequest request) {
    // 请求方法（GET/POST...）
    RequestMethodsRequestCondition methods = this.methodsCondition.getMatchingCondition(request);
    if (methods == null) return null;                 // 方法不对，直接淘汰
    // 请求参数（params = "..."）
    ParamsRequestCondition params = this.paramsCondition.getMatchingCondition(request);
    if (params == null) return null;
    // 请求头（headers = "..."）
    HeadersRequestCondition headers = this.headersCondition.getMatchingCondition(request);
    if (headers == null) return null;
    // 请求体类型（consumes）
    ConsumesRequestCondition consumes = this.consumesCondition.getMatchingCondition(request);
    if (consumes == null) return null;
    // 响应体类型（produces）
    ProducesRequestCondition produces = this.producesCondition.getMatchingCondition(request);
    if (produces == null) return null;
    // URL 路径（patterns）
    PatternsRequestCondition patterns = this.patternsCondition.getMatchingCondition(request);
    if (patterns == null) return null;
    // 全部通过 → 生成一个"更具体"的匹配结果（条件越少越精确，留给比较器排序）
    return new RequestMappingInfo(this.name, patterns, methods, params, headers, consumes, produces, this.customCondition);
}
```

> 这解释了为什么 `@GetMapping("/user")` 和 `@GetMapping` 同时存在时，前者优先——匹配条件越具体，排序时越靠前。

## MappingRegistry 里为什么有好几张表

注册表不是简单一张 Map，而是按"匹配方式"分了几个视角：

```java
class MappingRegistry {
    // Mapping → 注册信息（HandlerMethod、directUrls、name 等）
    private final Map<T, MappingRegistration<T>> registry = new HashMap<>();
    // Mapping → HandlerMethod，真正查到 Handler 用的就是它
    private final Map<T, HandlerMethod> mappingLookup = new LinkedHashMap<>();
    // 直接路径 → Mapping，用于 lookupHandlerMethod 里的"直接匹配"快速路径
    private final MultiValueMap<String, T> pathLookup = new LinkedMultiValueMap<>();
    // name → Mapping，仅按名字显式查找时用，不参与 URL 匹配
    private final Map<String, List<T>> nameLookup = new HashMap<>();
    // 读写锁：注册是写、匹配是读，读多写少，用读写锁最合适
    private final ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
}
```

三者分工：

- `pathLookup`：让"直接路径"能 O(1) 命中，是性能优化的快路径；
- `mappingLookup`：全量匹配时的数据源，也是最终返回 HandlerMethod 的地方；
- `nameLookup`：只服务于"按名字查"，**不要**把它当成 URL 匹配用的表（这是常见误解）。

注册在启动扫描时完成：

```java
public void register(T mapping, Object handler, Method method) {
    this.readWriteLock.writeLock().lock();          // 注册走写锁
    try {
        HandlerMethod handlerMethod = createHandlerMethod(handler, method);
        this.mappingLookup.put(mapping, handlerMethod);
        // 如果这个 Mapping 能给出"直接路径"，就顺手记进 pathLookup，加速后续匹配
        List<String> directUrls = getDirectUrls(mapping);
        for (String url : directUrls) {
            this.pathLookup.add(url, mapping);
        }
        this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));
    } finally {
        this.readWriteLock.writeLock().unlock();
    }
}
```

至此，`getHandlerInternal` 返回一个 **`HandlerMethod`**——它是对「Controller Bean + 目标方法」的包装，记下了"哪个对象上的哪个方法"。

## HandlerExecutionChain 是什么

回到 `AbstractHandlerMapping.getHandler`，它最后把 Handler 交给 `getHandlerExecutionChain`，包装成 **`HandlerExecutionChain`**：把 Handler 和匹配到的拦截器链绑在一起。

> 所以 `getHandler` 返回的从来不是裸 Handler，而是一条"处理器 + 拦截器"的执行链——拦截器就是在 `DispatcherServlet` 里按这条链触发的。

# 第二站：为什么还要 HandlerAdapter

拿到的 Handler 形态不止一种：可能是 `@RequestMapping` 标注的方法（`HandlerMethod`），也可能是实现了 `Controller` 接口的 Bean。它们的调用方式不一样，`DispatcherServlet` 却只想要一个统一的 `handle()` 入口——**这就是 `HandlerAdapter` 存在的理由：适配器模式，把不同形态的 Handler 统一成同一种调用方式。**

挑适配器的逻辑很简单，谁 `supports` 就用谁：

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        for (HandlerAdapter adapter : this.handlerAdapters) {
            if (adapter.supports(handler)) {     // 按 Handler 类型挑选
                return adapter;
            }
        }
    }
    throw new ServletException("No adapter for handler [" + handler + "]");
}
```

最直观的是 `SimpleControllerHandlerAdapter`（处理实现 `Controller` 接口的 Handler）：

```java
public class SimpleControllerHandlerAdapter implements HandlerAdapter {

    @Override
    public boolean supports(Object handler) {
        return (handler instanceof Controller);          // 只认 Controller 类型
    }

    @Override
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        return ((Controller) handler).handleRequest(request, response);   // 直接调
    }
}
```

而 `@RequestMapping` 方法由 **`RequestMappingHandlerAdapter`** 处理——参数解析、返回值处理都在它里面。

> 误区提醒：`HandlerAdapter` 是**适配器**，不是"执行器"。真正的业务执行发生在 Handler 内部，Adapter 只负责"把不同 Handler 统一调起来"。

# 拦截器链是怎么串起来的

前面 `doDispatch` 调了 `mappedHandler.applyPreHandle(...)`，它就是 `HandlerExecutionChain` 上的方法：

```java
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
    for (int i = 0; i < this.interceptorList.size(); i++) {     // 正序执行 preHandle
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        if (!interceptor.preHandle(request, response, this.handler)) {
            triggerAfterCompletion(request, response, null);    // 一旦中断，立刻触发已执行拦截器的 afterCompletion
            return false;
        }
        this.interceptorIndex = i;   // 记下执行到哪，供 afterCompletion 逆序回滚
    }
    return true;                     // 全部通过，继续往下走
}
```

对应的顺序：`preHandle` **正序**、`postHandle` **逆序**、`afterCompletion` **逆序**（且异常也会执行）。

> 限于篇幅，`postHandle` / `afterCompletion` 的源码留给读者，都是同样的"反向遍历"套路。

# 第三站：RequestMappingHandlerAdapter 真正执行

`ha.handle(...)` 最终走到 `RequestMappingHandlerAdapter.invokeHandlerMethod`（去掉了异步、CORS 分支）：

```java
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

        // 把 HandlerMethod 包装成"可被调用的方法"
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        // 装上参数解析器和返回值处理器——这是 SpringMVC 参数绑定能力的来源
        if (this.argumentResolvers != null) {
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        if (this.returnValueHandlers != null) {
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        invocableMethod.setDataBinderFactory(binderFactory);
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

        ModelAndViewContainer mavContainer = new ModelAndViewContainer();
        mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);

        // 关键：执行方法 + 处理返回值
        invocableMethod.invokeAndHandle(webRequest, mavContainer);

        return getModelAndView(mavContainer, modelFactory, webRequest);
    } finally {
        webRequest.requestCompleted();
    }
}
```

## 参数是怎么绑上来的

核心在 `InvocableHandlerMethod.invokeForRequest` → `getMethodArgumentValues`：**逐个参数找解析器，把请求里的数据绑成方法参数**。

```java
@Nullable
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {
    Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);   // 先把参数都算出来
    return doInvoke(args);                                                          // 再反射调用方法
}

protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    MethodParameter[] parameters = getMethodParameters();
    Object[] args = new Object[parameters.length];
    for (int i = 0; i < parameters.length; i++) {
        MethodParameter parameter = parameters[i];
        parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
        args[i] = findProvidedArgument(parameter, providedArgs);
        if (args[i] != null) {
            continue;
        }
        // 有没有解析器支持这个参数？（@RequestParam / @PathVariable / @RequestBody ...）
        if (!this.resolvers.supportsParameter(parameter)) {
            throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
        }
        try {
            // 交给对应的解析器算出参数值
            args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
        } catch (Exception ex) {
            // ... 记录日志后抛出
        }
    }
    return args;
}
```

"找解析器"的逻辑在 `HandlerMethodArgumentResolverComposite` 里，注意它用了**缓存**：

```java
@Override
public boolean supportsParameter(MethodParameter parameter) {
    return getArgumentResolver(parameter) != null;
}

@Nullable
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);   // 先查缓存
    if (result == null) {
        for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {         // 逐个试
            if (resolver.supportsParameter(parameter)) {
                result = resolver;
                this.argumentResolverCache.put(parameter, result);   // 命中后缓存，避免每次遍历
                break;
            }
        }
    }
    return result;
}
```

所以一套 `argumentResolvers`（约 20+ 个）就是这样被选中的：`@RequestParam` 走 `RequestParamMethodArgumentResolver`、`@PathVariable` 走 `PathVariableMethodArgumentResolver`、`@RequestBody` 走 `RequestResponseBodyMethodProcessor`……

## 返回值是怎么处理的

`invokeAndHandle` 在调用完方法后，把返回值交给**返回值处理器**：

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {
    Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
    // ... 处理 @ResponseStatus / 返回 null 的边界情况
    // 交给返回值处理器，比如 @ResponseBody 由 RequestResponseBodyMethodProcessor 处理
    this.returnValueHandlers.handleReturnValue(
            returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
}
```

`@ResponseBody` 的处理器干脆利落地把响应写掉：

```java
// RequestResponseBodyMethodProcessor（同时是 @RequestBody 的解析器）
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
    mavContainer.setRequestHandled(true);       // 标记"响应已处理"，后面不再走视图渲染
    ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
    ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);
    // 用 HttpMessageConverter 把返回值序列化（如 JSON）写回响应体
    writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}
```

# 最后一站：响应是怎么出去的

回到 `doDispatch` 的 ⑥，`processDispatchResult` 决定"渲染视图"还是"什么都不做"：

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
        @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
        @Nullable Exception exception) throws Exception {

    // ... 如果前面抛了异常，这里会调 processHandlerException 得到异常视图的 mv

    if (mv != null && !mv.wasCleared()) {
        render(mv, request, response);              // 真去渲染视图（Thymeleaf / JSP ...）
    }
    // ...

    if (mappedHandler != null) {
        mappedHandler.triggerAfterCompletion(request, response, null);   // 拦截器 afterCompletion
    }
}
```

关键在 `getModelAndView` 里那句判断：`@ResponseBody` 已经通过 `setRequestHandled(true)` 把 `mavContainer` 标记为"已处理"，于是它**直接返回 `null`**，`processDispatchResult` 里 `mv != null` 不成立，就跳过了视图渲染。

```java
@Nullable
private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
        ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {
    modelFactory.updateModel(webRequest, mavContainer);
    if (mavContainer.isRequestHandled()) {          // @ResponseBody 走的就是这里
        return null;
    }
    // ... 否则构建 ModelAndView，交给视图解析器
}
```

**到这一步，一条请求就完整闭环了。**

# 小结

1. **一次请求的核心流程？** `getHandler` → `getHandlerAdapter` → `applyPreHandle` → `handle` → `applyPostHandle` → `processDispatchResult`，都在 `DispatcherServlet.doDispatch`。
2. **URL 怎么变成 HandlerMethod 的？** `RequestMappingHandlerMapping` 启动时把注解汇编成 `RequestMappingInfo` 存进 `MappingRegistry`；请求时 `getHandlerInternal` → `lookupHandlerMethod`，先按"直接路径"快路径匹配，不中再全量匹配，多个命中则用比较器排序取最优。
3. **`RequestMappingInfo` 匹配哪些条件？** 请求方法、params、headers、consumes、produces、URL patterns——**逐项过滤，任意一项不通过即淘汰**。
4. **MappingRegistry 为什么有好几张表？** `pathLookup` 用于快速直接匹配，`mappingLookup` 用于全量匹配并取 HandlerMethod，`nameLookup` 只服务按名字查找（不参与 URL 匹配）。
5. **为什么需要 HandlerAdapter？** Handler 形态不止一种（`HandlerMethod` / `Controller` Bean），适配器把它们的调用方式统一成 `handle()`。
6. **HandlerExecutionChain 是什么？** Handler 的二次包装，把 Handler 和匹配到的拦截器链绑在一起。
7. **参数是怎么绑上来的？** `InvocableHandlerMethod.getMethodArgumentValues` 逐个参数找 `HandlerMethodArgumentResolver`，由 `ArgumentResolverComposite` 选出并缓存对应解析器，再 `resolveArgument`。
8. **`@ResponseBody` 为什么不走视图解析？** `RequestResponseBodyMethodProcessor` 用消息转换器把返回值写回响应，并 `setRequestHandled(true)`；`getModelAndView` 据此返回 null，跳过视图渲染。
9. **拦截器的执行顺序？** `preHandle` 正序、`postHandle` 逆序、`afterCompletion` 逆序（异常也执行）。
10. **`HandlerAdapter` 是执行器吗？** 不是，它是适配器；真正的执行发生在 Handler 内部。

# 参考

- Spring 5.x 源码：`DispatcherServlet`、`AbstractHandlerMapping`、`AbstractHandlerMethodMapping`、`RequestMappingInfo`、`RequestMappingHandlerAdapter`、`InvocableHandlerMethod`
- [SpringIOC.md](../spring/SpringIOC.md)、[SpringAop代理.md](../spring/SpringAop代理.md)
