Spring Aware 是一组"回调接口"：Bean 实现某个 `Aware` 接口后，Spring 会在合适的时机把对应的容器能力**回传**给它——比如把 `ApplicationContext`、`BeanFactory`、`Environment` 塞给你。

一句话：**Aware 是让 Bean 感知容器的标准方式。** 它不是用来解耦的，恰恰相反，用了 Aware 就和 Spring 绑定了，所以只在确实需要访问容器时才用。

# 总览

回调发生在 Bean 生命周期的**属性填充之后、初始化之前**（见 [Spring生命周期.md](Spring生命周期.md)）。常见的 Aware 接口：

| 接口 | 回传给你 | 典型用途 |
|---|---|---|
| `BeanNameAware` | `beanName` | 拿到自己的名字 |
| `BeanClassLoaderAware` | `ClassLoader` | 加载类/资源 |
| `BeanFactoryAware` | `BeanFactory` | 取 Bean 定义、按需获取其它 Bean |
| `ApplicationContextAware` | `ApplicationContext` | 获取容器、发布事件、拿资源 |
| `EnvironmentAware` | `Environment` | 读配置、profile |
| `ResourceLoaderAware` | `ResourceLoader` | 加载资源文件 |
| `MessageSourceAware` | `MessageSource` | 国际化消息 |
| `ApplicationEventPublisherAware` | `ApplicationEventPublisher` | 发布自定义事件 |

![img.png](_assets/Aware.png)

# 回调是在哪里触发的

分两拨，别搞混：

- `BeanNameAware` / `BeanClassLoaderAware` / `BeanFactoryAware` 由 `AbstractAutowireCapableBeanFactory.invokeAwareMethods` 直接回调；
- 其余（`ApplicationContextAware`、`EnvironmentAware`、`ResourceLoaderAware`、`ApplicationEventPublisherAware`、`MessageSourceAware`）由 **`ApplicationContextAwareProcessor`** 这个 `BeanPostProcessor` 在**初始化前**回调。

```java
// AbstractAutowireCapableBeanFactory：直接处理这三个
private void invokeAwareMethods(String beanName, Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
                ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
            }
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```

```java
// ApplicationContextAwareProcessor（一个 BeanPostProcessor）：处理其余的 Aware
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
          bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
          bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
        return bean;
    }
    // ... 逐个判断并回调，比如：
    if (bean instanceof ApplicationContextAware) {
        ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
    }
    return bean;
}
```

所以顺序是：`invokeAwareMethods`（BeanName/ClassLoader/BeanFactory）→ `ApplicationContextAwareProcessor` 等后置处理器（其余 Aware）→ 初始化回调。

# 用法示例

```java
@Service
public class StudentService implements ApplicationContextAware {
    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;   // 容器会把 ApplicationContext 回传进来
    }
}
```

# 自定义 Aware

Aware 本身就是一套扩展机制：你可以自定义 `XxxAware` 接口，再配一个 `BeanPostProcessor` 去回调它，就能把任意容器能力注入给 Bean。

# 小结

1. **Aware 是什么？** 一组回调接口，让 Bean 感知/访问容器。
2. **回调时机？** 属性填充之后、初始化之前。
3. **分几拨触发？** `BeanNameAware`/`BeanClassLoaderAware`/`BeanFactoryAware` 由 `invokeAwareMethods` 直接触发；其余由 `ApplicationContextAwareProcessor`（BeanPostProcessor）触发。
4. **该不该用？** 只在确实需要访问容器时用，否则会和 Spring 强耦合。

# 参考

- Spring 5.x 源码：`Aware`、`ApplicationContextAwareProcessor`、`AbstractAutowireCapableBeanFactory.invokeAwareMethods`
- [Spring生命周期.md](Spring生命周期.md)、[基本概念.md](基本概念.md)
