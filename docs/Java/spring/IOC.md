##提出问题
如果是我们自己来设计spring的IOC模块,会从哪些方面考虑？
> IOC全名Inversion of Control控制反转,意味着所有的实例对象统统交给spring管理。

### First
正常创建对象-交给BeanFactory
![img.png](_assets/createObject.png)

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
![img.png](_assets/BeanFactoryRegister.png)

*从图上可以看到多次使用模板模式进行扩展, 父类定义method的参数跟返回,具体地子类重载实现细节*

#### bean的创建方式
控制反转,意味着由spring控制bean的生成,因此获取的类就是代理类。spring提供了两种方式创建动态代理类

参考demo实现
```java
public class User {
    private Long id;
    private String name;

    public User() {

    }
    public User(Long id, String name) {
        this.id = id;
        this.name = name;
    }
}
```
- jdk反射机制
```java
public static void main(String[] args) {
        User user = new User();
        Class<?> clazz =user.getClass();
        try {
            Constructor<?> constructorToUse = null;
            Constructor[] constructors = clazz.getDeclaredConstructors();
            for(Constructor ctr:constructors){
                if(ctr.getParameterTypes().length==2){
                    constructorToUse=ctr;
                }
            }
            //有参构造函数
            Object[] arg=new Object[]{1L,"XXX"};
            User user1 = (User) clazz.getDeclaredConstructor(constructorToUse.getParameterTypes()).newInstance(arg);
            System.out.println(user1.getId());
            //无参构造函数
//            User user2 = (User) clazz.getDeclaredConstructor().newInstance();
//            System.out.println(user2.getId());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

```  
- cglib动态代理
```java
public static void main(String[] args){
        User user=new User();
        Class<?> clazz=user.getClass();
        Constructor<?> constructorToUse=null;
        Constructor[]constructors=clazz.getDeclaredConstructors();
        for(Constructor ctr:constructors){
        if(ctr.getParameterTypes().length==2){
        constructorToUse=ctr;
        }
        }
        Object[]arg=new Object[]{1L,"XXX"};
        Enhancer enhancer=new Enhancer();
        enhancer.setSuperclass(user.getClass());
        enhancer.setCallback(new NoOp(){
            @Override
            public int hashCode(){
            return super.hashCode();
            }
        });
        User user1=(User)enhancer.create(constructorToUse.getParameterTypes(),arg);
        System.out.println(user1.getId());
        }
```

#### bean属性填充
代理的Object已创建,但具体的属性还没有填充。

AbstractAutowireCapableBeanFactory->createBean->doGetBean->populateBean->applyPropertyValues

```java
	  // Set our (possibly massaged) deep copy.
		try {
			bw.setPropertyValues(new MutablePropertyValues(deepCopy));
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Error setting property values", ex);
		}
```
bean已经创建好,属性已经填充完毕,但如何将bean注册到springFactory还没有看到

#### bean的注册实现

读取资源->装载->注册

XMLBeanDefinitionReader->doLoadBeanDefinitions(加载资源)->DefaultBeanDefinitionDocumentReader->doRegisterBeanDefinitions(注册)
```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        if (bdHolder != null) {
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
          重点// Register the final decorated instance.
        BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
        getReaderContext().error("Failed to register bean definition with name '" +
        bdHolder.getBeanName() + "'", ele, ex);
        }
          注册完成后,发送消息通知
           // Send registration event.
           getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
        }
        }
```
#### bean的扩展机制
当前bean的使用,不具备扩展性。于是spring针对bean的生命周期提供了扩展口
![img.png](_assets/beanLifeCycle.png)

BeanFactoryPostProcessor: 是由 Spring 框架组建提供的容器扩展机制，允许在 Bean 对象注册后但未实例化之前，对 Bean 的定义信息 BeanDefinition 执行修改操作

```java
Factory hook that allows for custom modification of an application context's bean definitions, adapting the bean property values of the context's underlying bean factory.
Useful for custom config files targeted at system administrators that override bean properties configured in the application context. See PropertyResourceConfigurer and its concrete implementations for out-of-the-box solutions that address such configuration needs.
A BeanFactoryPostProcessor may interact with and modify bean definitions, but never bean instances. Doing so may cause premature bean instantiation, violating the container and causing unintended side-effects. If bean instance interaction is required, consider implementing BeanPostProcessor instead.
Registration
An ApplicationContext auto-detects BeanFactoryPostProcessor beans in its bean definitions and applies them before any other beans get created. A BeanFactoryPostProcessor may also be registered programmatically with a ConfigurableApplicationContext.
Ordering
BeanFactoryPostProcessor beans that are autodetected in an ApplicationContext will be ordered according to org.springframework.core.PriorityOrdered and org.springframework.core.Ordered semantics. In contrast, BeanFactoryPostProcessor beans that are registered programmatically with a ConfigurableApplicationContext will be applied in the order of registration; any ordering semantics expressed through implementing the PriorityOrdered or Ordered interface will be ignored for programmatically registered post-processors. Furthermore, the @Order annotation is not taken into account for BeanFactoryPostProcessor beans.
Since:
06.07.2003
See Also:
BeanPostProcessor, PropertyResourceConfigurer
Author:
Juergen Hoeller, Sam Brannen
简单翻译:
        spring IoC 容器允许 BeanFactoryPostProcessor 在容器实例化之前读取 Bean 的定义（也称配置元数据），并可以修改它们。可定义多个 BeanFactoryPostProcessor ，通过设置 order 属性（需要 Order 接口，当前暂时没书写此逻辑）来确定各个BeanFactoryPostProcessor执行顺序。

        注册一个 BeanFactoryPostProcessor 实例需要定义一个 Java 类来实现 BeanFactoryPostProcessor 接口，并重写该接口的 postProcessorBeanFactory 方法。通过 beanFactory 可以获取 bean 的定义信息，并可以修改 bean 的定义信息。这点是和 BeanPostProcessor 最大区别.
@FunctionalInterface
public interface BeanFactoryPostProcessor {
    /**
     * Modify the application context's internal bean factory after its standard
     * initialization. All bean definitions will have been loaded, but no beans
     * will have been instantiated yet. This allows for overriding or adding
     * properties even to eager-initializing beans.
     * @param beanFactory the bean factory used by the application context
     * @throws org.springframework.beans.BeansException in case of errors
     */
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```

BeanPostProcessor: BeanPostProcessor是在Bean对象实例化之后修改 Bean 对象，也可以替换 Bean 对象。这部分与AOP有着密切的关系

```java
public interface BeanPostProcessor {

    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
    
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```
针对这两个的原理和使用场景,单开一节详细说.这里知道Spring提供了对应的钩子能进行如下的操作。

#### bean是钩子函数实现
- init-method
初始化操作,是在BeanPostProcessor之后,实例化之前调用。
```java
public interface InitializingBean {
    /**
     * Invoked by the containing {@code BeanFactory} after it has set all bean properties
     * and satisfied {@link BeanFactoryAware}, {@code ApplicationContextAware} etc.
     * <p>This method allows the bean instance to perform validation of its overall
     * configuration and final initialization when all bean properties have been set.
     * @throws Exception in the event of misconfiguration (such as failure to set an
     * essential property) or if initialization fails for any other reason
     */
    void afterPropertiesSet() throws Exception;

}
```
AbstractAutowireCapableBeanFactory(类)->initializeBean(方法)->invokeInitMethods(方法)
- destroy
```java
public interface DisposableBean {

	/**
	 * Invoked by the containing {@code BeanFactory} on destruction of a bean.
	 * @throws Exception in case of shutdown errors. Exceptions will get logged
	 * but not rethrown to allow other beans to release their resources as well.
	 */
	void destroy() throws Exception;

}
```
AbstractApplicationContext注册钩子函数,在方法结束后会执行destroy方法。
```java
@Override
	public void registerShutdownHook() {
		if (this.shutdownHook == null) {
			// No shutdown hook registered yet.
			this.shutdownHook = new Thread(SHUTDOWN_HOOK_THREAD_NAME) {
				@Override
				public void run() {
					synchronized (startupShutdownMonitor) {
						doClose();
					}
				}
			};
			Runtime.getRuntime().addShutdownHook(this.shutdownHook);
		}
	}
```











