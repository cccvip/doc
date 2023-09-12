依赖注入是一个非常抽象的概念,那我们看看chatgpt是如何解释这个词的呢
Spring的依赖注入（Dependency Injection，简称DI）是一种设计模式，它允许对象之间的依赖关系由容器来管理和注入，而不是由对象自己创建和管理。依赖注入的核心思想是将对象之间的依赖关系解耦，使得代码更加灵活、可维护和可测试。

在Spring中，依赖注入通过容器自动将对象之间的依赖关系注入到对象中，而不需要手动创建和管理这些对象。通过使用注解或配置文件，我们可以告诉Spring容器哪些对象需要依赖注入，以及如何注入这些依赖。

依赖注入的优势包括：
- 降低了对象之间的耦合度，使得代码更加模块化和可扩展。
- 提高了代码的可测试性，可以更方便地进行单元测试。
- 提高了代码的可读性和可维护性，将对象之间的依赖关系集中管理，使得代码更加清晰。
总的来说，Spring的依赖注入是一种通过容器自动管理对象之间依赖关系的机制，它能够提供更加灵活、可维护和可测试的代码。

以@Autowired注解为例,AutowiredAnnotationBeanPostProcessor为注解的实现类,@Autowired作用范围可以是方法,构造方法,属性上
````java
/**
 * AutowiredAnnotationBeanPostProcessor
 */
@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Autowired {
	boolean required() default true;
}
````
我们得思考一件事要实现此功能,需要完成怎样一件事。
1. 我们得知道Bean中,哪些属性需要注入
2. 这些属性以一种什么样的方式注入(byName、byType、构造方法注入)
```java
public class AutowiredAnnotationBeanPostProcessor implements SmartInstantiationAwareBeanPostProcessor,
		MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware {
            
    //寻找需要注入的属性或者方法等
    @Override
    public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
        InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
        metadata.checkConfigMembers(beanDefinition);
    }

    //赋值    
    @Override
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
		InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
		try {
			metadata.inject(bean, beanName, pvs);
		}
		catch (BeanCreationException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
		}
		return pvs;
	}
}
```
注入点的执行流程
1. 遍历当前类的所有的属性字段Field
2. 查看字段上是否存在@Autowired、@Value、@Inject中的其中任意一个，存在则认为该字段是一个注入点
3. 如果字段是static的，则不进行注入
4. 获取@Autowired中的required属性的值
5. 将字段信息构造成一个AutowiredFieldElement对象，作为一个注入点对象添加到currElements集合中。
6. 遍历当前类的所有方法Method
7. 判断当前Method是否是桥接方法(主要用途是支持泛型)，如果是找到原方法
8. 查看方法上是否存在@Autowired、@Value、@Inject中的其中任意一个，存在则认为该方法是一个注入点
9. 如果方法是static的，则不进行注入
10. 获取@Autowired中的required属性的值
11. 将方法信息构造成一个AutowiredMethodElement对象，作为一个注入点对象添加到currElements集合中。
12. 遍历完当前类的字段和方法后，将遍历父类的，直到没有父类。
13. 最后将currElements集合封装成一个InjectionMetadata对象，作为当前Bean对于的注入点集合对象，并缓存。


## 注入点注入

### 字段注入

1. 遍历所有的AutowiredFieldElement对象。
2. 将对应的字段封装为DependencyDescriptor对象。
3. 调用BeanFactory的resolveDependency()方法，传入DependencyDescriptor对象，进行依赖查找，找到当前字段所匹配的Bean对象。
4. 将DependencyDescriptor对象和所找到的结果对象beanName封装成一个ShortcutDependencyDescriptor对象作为缓存，比如如果当前Bean是原型Bean，那么下次再来创建该Bean时，就可以直接拿缓存的结果对象beanName去BeanFactory中去那bean对象了，不用再次进行查找了
5. 利用反射将结果对象赋值给字段。


### Set方法注入
1. 遍历所有的AutowiredMethodElement对象
2. 遍历将对应的方法的参数，将每个参数封装成MethodParameter对象
3. 将MethodParameter对象封装为DependencyDescriptor对象
4. 调用BeanFactory的resolveDependency()方法，传入DependencyDescriptor对象，进行依赖查找，找到当前方法参数所匹配的Bean对象。
5. 将DependencyDescriptor对象和所找到的结果对象beanName封装成一个ShortcutDependencyDescriptor对象作为缓存，比如如果当前Bean是原型Bean，那么下次再来创建该Bean时，就可以直接拿缓存的结果对象beanName去BeanFactory中去那bean对象了，不用再次进行查找了
6. 利用反射将找到的所有结果对象传给当前方法，并执行。







