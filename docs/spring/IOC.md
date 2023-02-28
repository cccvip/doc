##提出问题
如果是我们自己来设计spring的IOC模块,会从哪些方面考虑？
> IOC全名Inversion of Control控制反转,意味着所有的实例对象统统交给spring管理。

### First
正常创建对象-交给BeanFactory
![img.png](createObject.png)

```java

/**
 * 简介：BeanDefinition描述了bean的实例属性以及构造方法,具体细节可以参考具体的实现类
 * A BeanDefinition describes a bean instance, which has property values,
 * constructor argument values, and further information supplied by
 * concrete implementations.
 * 重点：BeanDefinition是一个基础接口,最主要的目的是允许BeanFactoryPostProcessor修改属性值
 * <p>This is just a minimal interface: The main intention is to allow a
 * {@link BeanFactoryPostProcessor} to introspect and modify property values
 * and other bean metadata.
 */
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
 //省略代码
}
```
### Bean工厂
类的依赖关系图,理清关系脉络。
![img.png](BeanFactoryRegister.png)

*从图上可以看到多次使用模板模式进行扩展, 父类定义method的参数跟返回,具体地子类重载实现细节*

#### class的创建方式
要想动态的生成bean对象,也就是构造函数实例化。常见的有哪两种方式呢？
1 jdk反射机制
2 cglib动态代理








