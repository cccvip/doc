依赖注入（Dependency Injection，DI）是个很抽象的词。一句话说：**对象之间的依赖关系，不由对象自己创建，而由容器注入**。你只声明"我需要一个 X"，容器负责把 X 造好、塞进来。

Spring 的注入靠注解 + 反射。这篇以 `@Autowired` 为例，把"注入点怎么找、依赖怎么塞"这条链走一遍。

> 前面 [Spring生命周期.md](Spring生命周期.md) 讲到"属性填充"这一步——`@Autowired` 的真正实现就在这里。

# 总览

`@Autowired` 的处理者是 **`AutowiredAnnotationBeanPostProcessor`**，它作用在 Bean 的属性填充阶段。要实现自动注入，其实就两件事：

1. **找出注入点**：Bean 里哪些字段/方法需要注入？
2. **执行注入**：这些依赖按什么方式（byName / byType / 构造器）塞进去？

`@Autowired` 可以作用在字段、方法、构造方法、参数上：

```java
@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Autowired {
    boolean required() default true;     // 找不到依赖时是否报错
}
```

# 谁来处理：AutowiredAnnotationBeanPostProcessor

它实现了多个接口，关键的是 `SmartInstantiationAwareBeanPostProcessor` 和 `MergedBeanDefinitionPostProcessor`：

```java
public class AutowiredAnnotationBeanPostProcessor implements SmartInstantiationAwareBeanPostProcessor,
        MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware {

    // ① 在属性赋值前，收集注入点
    @Override
    public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
        InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
        metadata.checkConfigMembers(beanDefinition);
    }

    // ② 真正的注入：把依赖塞进去
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

一个负责**找**（`postProcessMergedBeanDefinition`），一个负责**塞**（`postProcessProperties`）。

# 注入点是怎么找出来的

`findAutowiringMetadata` 收集注入点的过程：

1. 遍历当前类的**所有字段** `Field`；
2. 字段上有 `@Autowired` / `@Value` / `@Inject` 中任意一个，就算一个注入点；
3. 如果字段是 `static` 的，跳过；
4. 读取 `@Autowired` 的 `required` 属性；
5. 把字段信息封成 `AutowiredFieldElement`，加入 `currElements`；
6. 遍历当前类的**所有方法** `Method`；
7. 若是桥接方法（桥接方法主要用于支持泛型），先找到原方法；
8. 方法上有上述三个注解之一，就算一个注入点；
9. 方法如果是 `static` 的，跳过；
10. 读取 `required`；
11. 把方法信息封成 `AutowiredMethodElement`，加入 `currElements`；
12. 当前类的字段和方法处理完，**继续处理父类**，直到没有父类；
13. 最后把 `currElements` 封装成 `InjectionMetadata`，作为当前 Bean 的注入点集合，并**缓存**。

# 依赖是怎么塞进去的

## 字段注入

1. 遍历所有 `AutowiredFieldElement`；
2. 把字段封装成 `DependencyDescriptor`；
3. 调用 BeanFactory 的 **`resolveDependency()`** 做依赖查找，找到匹配的 Bean；
4. 把 `DependencyDescriptor` 和查到的 Bean 封装成缓存（`ShortcutDependencyDescriptor`）——如果当前是原型 Bean，下次创建时可直接用缓存，省去再次查找；
5. 用**反射**把结果赋给字段。

## Setter 方法注入

1. 遍历所有 `AutowiredMethodElement`；
2. 遍历方法参数，每个参数封装成 `MethodParameter`；
3. `MethodParameter` 再封装成 `DependencyDescriptor`；
4. 调用 `resolveDependency()` 找到匹配的 Bean；
5. 同样做结果缓存；
6. 用**反射**把找到的对象作为参数传给方法并执行。

## 构造器注入

构造器注入的对象在**实例化阶段**就要准备好（不像字段注入在属性填充阶段），所以它和循环依赖的关系更微妙——构造器的循环依赖 Spring 无法解决，原因见 [Spring循环依赖.md](Spring循环依赖.md)。

# 小结

1. **`@Autowired` 的处理者是谁？** `AutowiredAnnotationBeanPostProcessor`，作用在属性填充阶段。
2. **它分两步做什么？** `postProcessMergedBeanDefinition` 收集注入点，`postProcessProperties` 执行注入。
3. **哪些算注入点？** 带 `@Autowired` / `@Value` / `@Inject` 的非 `static` 字段和方法。
4. **注入点范围？** 先本类，再逐级往上找父类。
5. **怎么找到依赖？** 把字段/参数封装成 `DependencyDescriptor`，交给 `BeanFactory.resolveDependency()` 按类型（必要时按名）查找。
6. **怎么赋值？** 反射：字段用 `Field.set`，方法用 `Method.invoke`。

# 参考

- Spring 5.x 源码：`AutowiredAnnotationBeanPostProcessor`、`DefaultListableBeanFactory.resolveDependency`
- [Spring生命周期.md](Spring生命周期.md)、[Spring循环依赖.md](Spring循环依赖.md)
