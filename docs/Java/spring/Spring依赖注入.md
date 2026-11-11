`@Autowired` 几乎是每个 Spring 项目都会用到的注解，但很多人只停留在"按类型注入"这一层：它到底在哪一步生效、同类型多个 Bean 时凭什么选这一个、为什么 `static` 字段注不进去，这些问题其实都能在源码里找到明确答案。

这篇按"先找注入点、再解析依赖、最后反射赋值"的顺序，把 `AutowiredAnnotationBeanPostProcessor` 的关键方法走一遍。如果只为面试，可以直接滑到最后的**小结**。

# 总览

**一句话：`@Autowired` 只做两件事——先找到注入点，再按类型解析依赖并反射赋值。**

处理者是 `AutowiredAnnotationBeanPostProcessor`，它实现 `SmartInstantiationAwareBeanPostProcessor` 和 `MergedBeanDefinitionPostProcessor`，在 Bean 的属性填充阶段工作。整体链路：

```text
AutowiredAnnotationBeanPostProcessor
  ├─ postProcessMergedBeanDefinition   → 找注入点，缓存到 InjectionMetadata
  └─ postProcessProperties             → 执行注入
        └─ resolveDependency
              └─ doResolveDependency
                    ├─ findAutowireCandidates      按类型找候选
                    └─ determineAutowireCandidate  多个候选时选一个
```

> 注入发生在 Bean 生命周期的"属性填充"阶段，位置见 [Spring生命周期.md](Spring生命周期.md) 的 `populateBean`。

# 关键概念

## 什么是依赖注入

依赖注入（Dependency Injection，DI）说的是：一个类需要用到另一个类时，不在自己内部 `new`，而是由外部把对象创建好、再"注入"进来。这里的"依赖"，就是当前类需要用到的其他对象。

以 `OrderService` 和 `OrderDao` 为例：

```java
// 不用 DI：OrderService 自己创建依赖
class OrderService {
    void create() {
        OrderDao dao = new OrderDao();   // 和具体实现绑死，换实现、写测试都麻烦
        dao.insert();
    }
}

// 用 DI：依赖由外部注入
class OrderService {
    private OrderDao dao;                // 只声明，不创建
    OrderService(OrderDao dao) {
        this.dao = dao;                  // 构造器注入
    }
}
```

这样做的好处：`OrderService` 不需要知道 `OrderDao` 怎么创建、用哪个实现；换实现或写单元测试时，直接传一个实现或假对象进去就行；对象的创建和生命周期也统一交给容器管理。

## 三种注入方式

| 方式 | 发生阶段 | 特点 |
|---|---|---|
| 字段注入 | 属性填充 | 写法最短，但依赖关系不直观，单元测试不好替换 |
| setter 注入 | 属性填充 | 依赖可选，可以在运行时重新设置 |
| 构造器注入 | 实例化 | 依赖创建时就必须就位，不可变，推荐优先使用 |

## @Autowired / @Resource / @Inject

| | `@Autowired` | `@Resource` | `@Inject` |
|---|---|---|---|
| 来源 | Spring | JSR-250 | JSR-330 |
| 默认策略 | 按类型（byType） | 按名字（byName），找不到再按类型 | 按类型（byType） |
| 指定名字 | `@Qualifier` | `name` 属性 | `@Named` |
| 必需性 | `required=true` | 默认必需 | 默认必需 |
| 处理者 | `AutowiredAnnotationBeanPostProcessor` | `CommonAnnotationBeanPostProcessor` | `AutowiredAnnotationBeanPostProcessor` |

# 源码分析

## 入口：两个回调方法

`AutowiredAnnotationBeanPostProcessor` 的两个关键方法，一个负责找，一个负责注入：

```java
public class AutowiredAnnotationBeanPostProcessor implements SmartInstantiationAwareBeanPostProcessor,
        MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware {

    // ① 属性填充前：扫描注入点，缓存到 InjectionMetadata
    @Override
    public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
        InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
        metadata.checkConfigMembers(beanDefinition);
    }

    // ② 属性填充时：执行注入
    @Override
    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
        InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
        try {
            metadata.inject(bean, beanName, pvs);
        } catch (BeanCreationException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
        }
        return pvs;
    }
}
```

## 找注入点：findAutowiringMetadata

`findAutowiringMetadata` 先查缓存，没有才扫描。缓存 key 是 beanName（没有就用类名）：

```java
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
    String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
    InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);

    if (InjectionMetadata.needsRefresh(metadata, clazz)) {
        synchronized (this.injectionMetadataCache) {
            metadata = this.injectionMetadataCache.get(cacheKey);
            if (InjectionMetadata.needsRefresh(metadata, clazz)) {
                if (metadata != null) {
                    metadata.clear(pvs);
                }
                // 真正扫描：字段 + 方法，并沿继承链往上找
                metadata = buildAutowiringMetadata(clazz);
                this.injectionMetadataCache.put(cacheKey, metadata);
            }
        }
    }
    return metadata;
}
```

真正扫描的是 `buildAutowiringMetadata`：

```java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
    List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
    Class<?> targetClass = clazz;

    do {
        final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();

        // 1. 扫当前类的字段
        ReflectionUtils.doWithLocalFields(targetClass, field -> {
            MergedAnnotation<?> ann = findAutowiredAnnotation(field);   // @Autowired / @Value / @Inject
            if (ann != null) {
                if (Modifier.isStatic(field.getModifiers())) {
                    return;                                             // static 字段不注入
                }
                boolean required = determineRequiredStatus(ann);
                currElements.add(new AutowiredFieldElement(field, required));
            }
        });

        // 2. 扫当前类的方法
        ReflectionUtils.doWithLocalMethods(targetClass, method -> {
            // 桥接方法要找到原方法，否则泛型方法会漏掉
            Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
            if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                return;
            }
            MergedAnnotation<?> ann = findAutowiredAnnotation(bridgedMethod);
            if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                if (Modifier.isStatic(method.getModifiers())) {
                    return;                                             // static 方法不注入
                }
                if (method.getParameterCount() == 0) {
                    return;                                             // 无参方法没东西可注入
                }
                boolean required = determineRequiredStatus(ann);
                PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                currElements.add(new AutowiredMethodElement(method, required, pd));
            }
        });

        // 3. 当前类的元素插到列表最前面；父类继续往上扫
        elements.addAll(0, currElements);
        targetClass = targetClass.getSuperclass();
    } while (targetClass != null && targetClass != Object.class);

    return InjectionMetadata.forElements(elements, clazz);
}
```

几个容易忽略的点：

- `@Autowired` / `@Value` / `@Inject` 三者都算注入点；
- `static` 字段和 `static` 方法直接跳过；
- 方法参数个数为 0 的跳过；
- 桥接方法要先找到原方法，否则泛型方法会漏掉；
- **扫描顺序是从本类往父类走**，但 `elements.addAll(0, currElements)` 是把当前类插到最前面，所以最终列表里**父类的注入点排在子类前面**，注入时也先处理父类。

## 解析依赖：resolveDependency

注入点收集完，`InjectionMetadata.inject` 会遍历每个元素调用 `inject`。最终都会走到 `BeanFactory.resolveDependency`：

```java
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
        @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

    descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());

    // ① 特殊类型：Optional / ObjectFactory / ObjectProvider / JSR-330 Provider
    if (Optional.class == descriptor.getDependencyType()) {
        return createOptionalDependency(descriptor, requestingBeanName);
    } else if (ObjectFactory.class == descriptor.getDependencyType() ||
            ObjectProvider.class == descriptor.getDependencyType()) {
        return new DependencyObjectProvider(descriptor, requestingBeanName);
    } else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
        return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
    } else {
        // ② @Lazy：返回一个代理，第一次调用时才真正解析
        Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(descriptor, requestingBeanName);
        if (result == null) {
            result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
        }
        return result;
    }
}
```

普通依赖走 `doResolveDependency`，核心逻辑是"按类型找候选 → 多个时选一个"：

```java
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
        @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

    Class<?> type = descriptor.getDependencyType();

    // ① @Value：先看有没有建议值（配置文件里的值）
    Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
    if (value != null) {
        // 解析占位符 + 类型转换，略
        return value;
    }

    // ② 数组 / Collection / Map：一次性注入多个候选
    Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
    if (multipleBeans != null) {
        return multipleBeans;
    }

    // ③ 按类型找所有候选 Bean
    Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
    if (matchingBeans.isEmpty()) {
        if (isRequired(descriptor)) {
            raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
        }
        return null;
    }

    String autowiredBeanName;
    Object instance;

    if (matchingBeans.size() > 1) {
        // ④ 多个候选：按 @Primary → @Priority → 名字 依次选
        autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
        if (autowiredBeanName == null) {
            return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
        }
        instance = matchingBeans.get(autowiredBeanName);
    } else {
        // 只有一个候选，直接用
        Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
        autowiredBeanName = entry.getKey();
        instance = entry.getValue();
    }

    if (autowiredBeanNames != null) {
        autowiredBeanNames.add(autowiredBeanName);
    }
    return (instance != null ? instance : descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans));
}
```

### 按类型找候选：findAutowireCandidates

```java
protected Map<String, Object> findAutowireCandidates(@Nullable String beanName, Class<?> requiredType,
        DependencyDescriptor descriptor) {

    // 1. 找出容器里所有该类型的 Bean 名字（包括父容器）
    String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
            this, requiredType, true, descriptor.isEager());

    Map<String, Object> result = CollectionUtils.newLinkedHashMap(candidateNames.length);

    // 2. 过滤：排除自己、排除不满足 @Qualifier 等条件的
    for (String candidate : candidateNames) {
        if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
            addCandidateEntry(result, candidate, descriptor, requiredType);
        }
    }

    // 3. 没有候选时，再考虑自引用（A 注入 A 的场景）
    if (result.isEmpty()) {
        // ...
    }
    return result;
}
```

注意 `isAutowireCandidate(candidate, descriptor)` 里已经处理了 `@Qualifier`：`QualifierAnnotationAutowireCandidateResolver` 会先按 qualifier 过滤一遍。所以 `@Qualifier` 不是在最后"选"的时候才生效，而是**提前缩小候选范围**。

### 多个候选怎么选：determineAutowireCandidate

```java
protected String determineAutowireCandidate(Map<String, Object> candidates, DependencyDescriptor descriptor) {
    Class<?> requiredType = descriptor.getDependencyType();

    // ① 先看 @Primary
    String primaryCandidate = determinePrimaryCandidate(candidates, requiredType);
    if (primaryCandidate != null) {
        return primaryCandidate;
    }

    // ② 再看 @Priority（javax.annotation.Priority）
    String priorityCandidate = determineHighestPriorityCandidate(candidates, requiredType);
    if (priorityCandidate != null) {
        return priorityCandidate;
    }

    // ③ 最后按名字匹配：字段名 / 参数名 与 Bean 名字一致
    for (Map.Entry<String, Object> entry : candidates.entrySet()) {
        String candidateName = entry.getKey();
        Object beanInstance = entry.getValue();
        if ((beanInstance != null && this.resolvableDependencies.containsValue(beanInstance)) ||
                matchesBeanName(candidateName, descriptor.getDependencyName())) {
            return candidateName;
        }
    }

    return null;   // 还选不出来，就抛 NoUniqueBeanDefinitionException
}
```

所以完整顺序是：

```text
@Qualifier 过滤候选  →  @Primary  →  @Priority  →  字段名/参数名匹配  →  报错 NoUniqueBeanDefinition
```

`@Qualifier` 在最前面，是因为它在 `findAutowireCandidates` 的 `isAutowireCandidate` 里就已经把候选筛掉了。

## 字段注入

`AutowiredFieldElement.inject` 把字段封装成 `DependencyDescriptor`，解析后反射赋值：

```java
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
    Field field = (Field) this.member;
    Object value;
    if (this.cached) {
        value = resolvedCachedArgument(beanName, this.cachedFieldValue);
    } else {
        value = resolveFieldValue(field, bean, beanName);
    }
    if (value != null) {
        ReflectionUtils.makeAccessible(field);
        field.set(bean, value);        // 反射赋值
    }
}

private Object resolveFieldValue(Field field, Object bean, @Nullable String beanName) {
    DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
    desc.setContainingClass(bean.getClass());

    Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
    Object value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, beanFactory.getTypeConverter());

    // 解析结果缓存起来：原型 Bean 下次创建时可以复用，不用重新查
    // ...
    return value;
}
```

## Setter / 方法注入

`AutowiredMethodElement` 把每个参数封装成 `DependencyDescriptor`，解析完后反射调用方法：

```java
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
    Method method = (Method) this.member;
    Object[] arguments = resolveMethodArguments(method, bean, beanName);
    if (arguments != null) {
        ReflectionUtils.makeAccessible(method);
        method.invoke(bean, arguments);   // 反射调用 setter / 普通方法
    }
}

private Object[] resolveMethodArguments(Method method, Object bean, @Nullable String beanName) {
    int argumentCount = method.getParameterCount();
    Object[] arguments = new Object[argumentCount];

    for (int i = 0; i < argumentCount; i++) {
        MethodParameter methodParam = new MethodParameter(method, i);
        DependencyDescriptor currDesc = new DependencyDescriptor(methodParam, this.required);
        currDesc.setContainingClass(bean.getClass());
        arguments[i] = beanFactory.resolveDependency(currDesc, beanName, null);
    }
    return arguments;
}
```

## 构造器注入

构造器注入不在属性填充阶段，而是在**实例化阶段**：`AutowiredAnnotationBeanPostProcessor` 实现 `SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors`，帮 `createBeanInstance` 挑构造器。

规则：

1. 只有一个构造器：直接用，不需要 `@Autowired`（Spring 4.3+）；
2. 多个构造器，其中一个带 `@Autowired`：用这个；
3. 多个构造器，都没有 `@Autowired`：用无参构造；
4. 没有无参构造又没有 `@Autowired`：启动报错。

构造器注入的依赖必须在对象实例化时就准备好，所以它和循环依赖的关系更紧——对象还没 new 出来，没法提前暴露，详见 [Spring循环依赖.md](Spring循环依赖.md)。

# 实战与常见坑

## 实战：同类型多个实现怎么选

假设有两个支付实现，`OrderService` 需要指定用微信支付：

```java
interface PayService {
    void pay();
}

@Service("aliPay")
class AliPayService implements PayService {
    public void pay() { /* 支付宝 */ }
}

@Service("wxPay")
class WxPayService implements PayService {
    public void pay() { /* 微信 */ }
}

@Service
class OrderService {
    @Autowired
    @Qualifier("wxPay")            // 没有这行会发生什么？
    private PayService payService;
}
```

对应到前面的源码：

1. `findAutowireCandidates` 按类型找到 `aliPay` 和 `wxPay` 两个候选；
2. `@Qualifier("wxPay")` 在 `isAutowireCandidate` 里先把 `aliPay` 过滤掉，只剩 `wxPay`；
3. 如果去掉 `@Qualifier`，候选有两个，`determineAutowireCandidate` 会依次看 `@Primary`、`@Priority`、字段名匹配；
4. 字段名 `payService` 和 `aliPay`、`wxPay` 都不匹配，最终抛 `NoUniqueBeanDefinitionException`。

解决办法就是源码里那几种：加 `@Qualifier`、给其中一个加 `@Primary`、或者把字段改成和 Bean 名一致。

## 常见坑

1. **同类型多个 Bean 却没指定**：抛 `NoUniqueBeanDefinitionException`，用 `@Primary`、`@Qualifier` 或字段名匹配解决；
2. **`static` 字段注入失败**：`buildAutowiringMetadata` 直接跳过 static 字段；
3. **父类字段先注入**：`elements.addAll(0, currElements)` 导致父类注入点排在子类前面，别假设子类先初始化；
4. **`@Autowired` 加到无参方法上没用**：参数个数为 0 会被跳过；
5. **`@Lazy` 能解决构造器循环依赖**：它注入的是代理，第一次调用才解析；
6. **`@Resource` 不是按类型优先**：默认按名字，名字对不上才退回按类型，和 `@Autowired` 行为不同。

# 小结

我一向不太喜欢写总结，因为该说的正文里都说了。不过考虑到很多读者是冲着面试来的，还是列成清单：

1. **`@Autowired` 的处理者是谁？** `AutowiredAnnotationBeanPostProcessor`，在属性填充阶段工作。
2. **它分两步做什么？** `postProcessMergedBeanDefinition` 收集注入点，`postProcessProperties` 执行注入。
3. **哪些算注入点？** 带 `@Autowired` / `@Value` / `@Inject` 的非 `static` 字段和方法。
4. **扫描顺序？** 从本类往父类扫；但最终列表父类在前，注入时先处理父类。
5. **依赖怎么找？** `resolveDependency` → `doResolveDependency` → `findAutowireCandidates` 按类型找候选。
6. **多个候选怎么选？** `@Qualifier` 先过滤 → `@Primary` → `@Priority` → 字段名/参数名匹配；都不行就报错。
7. **字段和方法怎么赋值？** 反射：字段 `Field.set`，方法 `Method.invoke`。
8. **构造器注入在哪一步？** 实例化阶段，由 `determineCandidateConstructors` 挑构造器完成。

# 参考

- [Spring Framework 官方文档：Annotation-based Container Configuration](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired.html)
- Spring 5.x 源码：`AutowiredAnnotationBeanPostProcessor`、`DefaultListableBeanFactory.resolveDependency`
- [Spring生命周期.md](Spring生命周期.md)、[Spring循环依赖.md](Spring循环依赖.md)
