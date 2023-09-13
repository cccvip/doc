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

## 为什么spring不使用一级缓存解决问题呢
![img.png](_assets/beanRecyle.png)

因为对象还有属性未被赋值,未被初始化。在spring的Bean生命周期中,bean存在实例化和初始化后的两种状态存在,也可称之为半成品化和成品化。如果只有一级缓存(一个Map),是无法解决bean生命周期两个状态bean共存的这个问题。

## 二级缓存能解决问题吗？
假想一下两个缓存 一个存成品,一个存半成品是否能解决问题? 如果不能 可能会出什么问题？
如果仅仅是为了解决问题,当然是可以。但是spring没有采用这种方法地原因是Aop特性,Aop的实现机制是在Bean最后一步,对Bean生成Aop-Target类。
如果只使用两级缓存,就会导致生成的Bean在实例化后就会创建,破坏了Bean生命周期

### Aop代理对象提前放入了三级缓存,没有经过初始化和属性填充,属性是什么时候被填充的呢？

Aop代理的是原始的bean,原始的bean在属性赋值的时候,保证了周期的完善。执行addSingletonFactory,将代理类移除二级缓存,加入一级缓存中

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


