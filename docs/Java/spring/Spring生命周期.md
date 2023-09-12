```java
public class App {
    @Test
    public void testApplication() {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext("com.carl.service");
        StudentService studentService = (StudentService) context.getBean("studentService");
        System.out.println(studentService);
    }
}
```

## 生成BeanDefinition
BeanDefinition定义了Bean的属性、scope、以及上下文信息等。
1. ClassPathBeanDefinitionScanner执行doScan方法,扫描basePackages下的.class文件
2. 遍历Resource数组
3. 使用MetadataReader解析Resource,MetadataReader默认实现为SimpleMetadataReader。
4. 匹配过滤metadataReader,利用MetadataReader进行excludeFilters和includeFilters，以及条件注解@Conditional的筛选
5. 生成ScannedGenericBeanDefinition
6. 增加筛选过滤,扫描到了一个Bean，将ScannedGenericBeanDefinition加入结果集

生成BeanDefinition后,就需要根据BeanDefinition定义的内容生成Bean,然后交给Spring工厂进行管理,大概流程如下
1. AbstractApplicationContext执行refresh()方法,
2. 在初始化之前,执行invokeBeanFactoryPostProcessors(beanFactory),可以修改BeanDefinition的属性定义等
- BeanDefinition属性可修改
3. 执行BeanFactoryPostProcessor的postProcessBeanFactory(beanFactory)方法
4. 注册BeanPostProcessor的在BeanFactory
- BeanDefinition实例化
5. 执行finishBeanFactoryInitialization(beanFactory),实例化所有非lazy加载的Bean
6. AbstractApplicationContext执行beanFactory.preInstantiateSingletons()
7. 核心实现在DefaultListableBeanFactory中

## preInstantiateSingletons执行流程
*分支流程(if-else较多),只关注核心实现即可*
1. 根据名字遍历BeanDefinition
2. 判断Bean是否已在三级缓存中存在(三级缓存后面解析原理),不在缓存中就执行createBean(beanName, mbd, args);
3. 类的实例化,第一步就是加载.class文件,如果beanClass属性的类型是Class，那么就直接返回，如果不是，则会根据类名进行加载（doResolveBeanClass方法所做的事情）
4. 实例化
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
	    //..
            // 提前实例化,返回一个代理Bean,
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
			//初始化创建真正的Bean
            Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
    }
    protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    		Object bean = null;
    		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
    			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
    				Class<?> targetType = determineTargetType(beanName, mbd);
    				if (targetType != null) {
                        // 提供一个提前初始化的时机，这里会直接返回一个实例对象
    					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
    					if (bean != null) {
                            // 如果提前初始化成功，则执行 postProcessAfterInitialization()方法
                            //注意:这里并不是applyBeanPostProcessorsAfterInstantiation,也就意味不会执行实例化的后置处理,相当于这里直接短路,流程到此为止
    						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
    					}
    				}
    			}
    			mbd.beforeInstantiationResolved = (bean != null);
    		}
    		return bean;
    	}
}
```

5.执行doCreateBean方法,生成Bean

5.1. 首先判断BeanDefinition中是否设置了Supplier，如果设置了则调用Supplier的get()得到对象。

5.2. 检查BeanDefinition中是否设置了factoryMethod,使用工厂方法创建对象

5.3  以上规则都没有,默认使用无参构造方法创建实例

6. BeanDefinition的后置处理    
Bean对象实例化出来之后，接下来就应该给对象的属性赋值了。在真正给属性赋值之前，Spring又提供了一个扩展点MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()

7. 注解的自动注入
后续章节补充

8. 属性解析
处理@Autowired、@Resource、@Value等注解，也是通过InstantiationAwareBeanPostProcessor.postProcessProperties()扩展点来实现的
9. 执行Aware
完成了属性赋值之后，Spring会执行一些回调，包括：
- BeanNameAware：回传beanName给bean对象。
- BeanClassLoaderAware：回传classLoader给bean对象。
- BeanFactoryAware：回传beanFactory给对象。

10.初始化前执行 BeanPostProcessor.postProcessBeforeInitialization()

11.初始化
- 查看当前Bean对象是否实现了InitializingBean接口，如果实现了就调用其afterPropertiesSet()方法
- 执行BeanDefinition中指定的初始化方法 

12.初始化后执行BeanPostProcessor.postProcessAfterInitialization()


13.Bean的注销机制

在Bean创建过程中，在初始化之后,有一个步骤会去判断当前创建的Bean是不是DisposableBean：

13.1. 当前Bean是否实现了DisposableBean接口 或者当前Bean是否实现了AutoCloseable接口

13.2. BeanDefinition中是否指定了destroyMethod

13.3. 调用DestructionAwareBeanPostProcessor.requiresDestruction(bean)进行判断
  a. ApplicationListenerDetector中直接使得ApplicationListener是DisposableBean
  b. InitDestroyAnnotationBeanPostProcessor中使得拥有@PreDestroy注解了的方法就是DisposableBean

13.4. 把符合上述任意一个条件的Bean适配成DisposableBeanAdapter对象，并存入disposableBeans中（一个LinkedHashMap）

























