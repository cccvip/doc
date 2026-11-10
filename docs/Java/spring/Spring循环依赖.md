## 循环依赖问题

# what
循环依赖问题（Circular Dependency）是指两个或多个模块或组件之间存在相互依赖关系，形成了一个闭环，造成程序无法正常执行或出现不可预料的行为。

# how

## 一级缓存能否解决循环依赖问题呢？
A依赖B B依赖A完全OK
```java
public class Singleton{

private static final Map<String,Object> singleMap = new ConcurrentHashMap<>();

    public static void main(String[] args) throws Exception {
        System.out.println(getBean(B.class).getA());
        System.out.println(getBean(A.class).getB());
    }
    
    private static <T> T getBean(Class<T> beanClass) throws Exception {
        String beanName = beanClass.getSimpleName().toLowerCase();
        if (singleMap.containsKey(beanName)) {
            return (T) singleMap.get(beanName);
        }
        // 实例化对象入缓存
        Object obj = beanClass.newInstance();
        singleMap.put(beanName, obj);
        // 属性填充补全对象
        Field[] fields = obj.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            Class<?> fieldClass = field.getType();
            String fieldBeanName = fieldClass.getSimpleName().toLowerCase();
            field.set(obj, singleMap.containsKey(fieldBeanName) ? singleMap.get(fieldBeanName) : getBean(fieldClass));
            field.setAccessible(false);
        }
        return (T) obj;
    }
}

class A {
    B b;

    public B getB() {
        return b;
    }

    public void setB(B b) {
        this.b = b;
    }
}

class B {
    A a;

    public A getA() {
        return a;
    }

    public void setA(A a) {
        this.a = a;
    }
}
```
```shell script
-----------------------------------------
打印结果 
com.xiao.A@4f3f5b24
com.xiao.B@15aeb7ab
```

## 为什么 spring 还要多级缓存？
![img.png](_assets/beanRecyle.png)

先澄清一点：上面的一级缓存 demo 对**普通字段注入**是能跑通的——它先把对象放进 Map 再填充属性，确实能解开 A 和 B 的循环。所以"一级缓存解决不了循环依赖"这个说法并不准确。

Spring 不用纯一级缓存，真正的原因是 **AOP 代理**：

- 循环依赖时，Spring 需要在对象还没初始化完成时就把引用暴露给对方；
- 但代理对象本应在**初始化之后**由 BeanPostProcessor 生成；
- 只有一个 Map 就无法区分"原始对象"和"最终代理对象"，也没法在正确时机把代理换进去。

于是才有了三级缓存的分工。

## 二级缓存能解决吗？

两个缓存（成品 + 半成品）对普通循环依赖确实够用。Spring 没有只用两级，还是因为 AOP：如果二级缓存里放的是原始对象，那么提前暴露出去的就是原始对象；等后续代理对象生成后，其它 Bean 里已经注入的引用可能仍指向原始对象，导致增强失效。

所以需要第三级缓存：放的是 `ObjectFactory`（Lambda），等到真正被依赖时才调用它，决定"要不要提前生成代理"，从而既解决循环依赖，又保证 AOP 的时机。

### 三级缓存里放的到底是什么？

三级缓存 `singletonFactories` 里放的不是代理对象，而是一个 `ObjectFactory`（Lambda：`() -> getEarlyBeanReference(beanName, mbd, bean)`）。它的作用是延迟决策：只有真的出现循环依赖、被别人来取时，才执行这个 Lambda；执行时会走 `getEarlyBeanReference`——如果该 Bean 需要 AOP，就在这里提前生成代理并放入二级缓存 `earlySingletonObjects`，否则直接返回原始对象。

属性填充仍然发生在原始对象上；等 Bean 完整初始化后，再把它（或代理）放入一级缓存 `singletonObjects`。`addSingletonFactory` 只是把工厂注册进三级缓存，不是"把代理类从二级移到一级"。

```shell script
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}
```

## spring为什么不能解决构造器的循环依赖？
如果使用构造器的循环依赖,则依赖对象就需要在构造函数中完成初始化。这时候就会出现问题,循坏依赖难以解决

## spring为什么不能解决prototype作用域的循环依赖？
此时生成的bean每次都是new的bean,并不会放在缓存中。但spring中循环依赖解决是通过缓存解决的

## 总结
![img.png](_assets/Totalcycle.png)

举一个例子
![img.png](_assets/example.png)

![img.png](_assets/exampleDetail.png)

至此，总结一下三级缓存：
1. singletonObjects：缓存经过了完整生命周期的bean
2. earlySingletonObjects：缓存未经过完整生命周期的bean，如果某个bean出现了循环依赖，就会提前把这个暂时未经过完整生命周期的bean放入earlySingletonObjects中，这个bean如果要经过AOP，那么就会把代理对象放入earlySingletonObjects中，否则就是把原始对象放入earlySingletonObjects，但是不管怎么样，就是是代理对象，代理对象所代理的原始对象也是没有经过完整生命周期的，所以放入earlySingletonObjects我们就可以统一认为是未经过完整生命周期的bean。
3. singletonFactories：缓存的是一个ObjectFactory，也就是一个Lambda表达式。在每个Bean的生成过程中，经过实例化得到一个原始对象后，都会提前基于原始对象暴露一个Lambda表达式，并保存到三级缓存中，这个Lambda表达式可能用到，也可能用不到，如果当前Bean没有出现循环依赖，那么这个Lambda表达式没用，当前bean按照自己的生命周期正常执行，执行完后直接把当前bean放入singletonObjects中，如果当前bean在依赖注入时发现出现了循环依赖（当前正在创建的bean被其他bean依赖了），则从三级缓存中拿到Lambda表达式，并执行Lambda表达式得到一个对象，并把得到的对象放入二级缓存（(如果当前Bean需要AOP，那么执行lambda表达式，得到就是对应的代理对象，如果无需AOP，则直接得到一个原始对象)）。
4. 其实还要一个缓存，就是earlyProxyReferences，它用来记录某个原始对象是否进行过AOP了。


