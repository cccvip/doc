最近在研究reqctor-netty源码,然后看到代码中使用java8的一些特性(响应式编程),简单聊一聊。


先说java8的函数式接口

### @FunctionalInterface
使用该注解,编译器会默认检查该interface下面是否只有一个抽象方法

可以增加多个static方法以及default方法,如果实现类继承接口,也不影响使用
```java
@FunctionalInterface
public interface A {
    void a();
    default void print() {
        System.out.println("xxxxx");
    }
}

public class AImpl implements A {

    @Override
    public void a() {
        print();
    }

}
```

### 常用函数实现接口

- Consumer<T>：消费型接口
> 抽象方法： void accept(T t)，接收一个参数进行消费，但无需返回结果。

- Supplier<T>: 供给型接口
> 抽象方法：T get()，无参数，有返回值。

- Function<T,R>: 函数型接口
> 抽象方法： R apply(T t)，传入一个参数，返回想要的结果

- Predicate<T> ： 断言型接口
> 抽象方法： boolean test(T t),传入一个参数，返回一个布尔值。

接口很多,省略其他接口参考java.util.function下的接口















