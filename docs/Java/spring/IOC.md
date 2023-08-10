# 前言

Spring IOC（Inversion of Control，控制反转）是Spring框架的一个核心特性。它是一种设计原则，通过该原则可以实现解耦和提高代码的可维护性。 Spring
IOC的核心思想是通过反转对象的创建和依赖关系的管理，将对象的创建和依赖关系的处理从应用程序代码中解耦出来，由Spring容器来负责管理。应用程序只需要声明需要依赖的对象，而不需要关心对象的创建和管理细节。

具体来说，应用程序通过配置文件或注解告诉Spring容器需要创建哪些对象，以及它们之间的依赖关系。Spring容器根据这些配置信息，在应用程序运行时动态地创建和管理对象，并自动解决对象之间的依赖关系。

通过使用Spring IOC，可以实现以下好处：

解耦：应用程序的各个模块之间通过接口进行交互，不再直接依赖具体的实现类，降低模块之间的耦合度。 可维护性：对象的创建和依赖关系的管理集中在Spring容器中，使得代码更容易理解、测试和维护。
可扩展性：通过配置文件或注解，可以方便地添加、修改或替换对象的实现，而不需要修改应用程序的源代码。

# 注解>XML

随着spring的时代发展,使用XML的方式声明Bean,似乎已在旧时代了。微服务概念的兴起,SpringBoot提倡约定大于配置。减少更多的配置文件,采取就开始使用注解,例如@Autowired, @Resource。

# 理解IOC

*一次声明,多次使用* (默认是单例)

举一个场景,基于MVC架构,controller调用service,service调用dao。

```java
import javax.annotation.Resource;

public class UserController {
    @Resource
    UserService userService;

    public void login() {
        userService.login();
    }
}

public class UserService {
    @Resource
    UserDao userDao;

    public void login() {
        userDao.login();
    }
}

//DAO 操作SQL
```

如果不使用@Resource,我们可能得这么干。每次使用需要new 对象,如果这么干,那么我们gc回收的频率得多快,影响使用。

```java
public class UserController {
    public void login() {
        UserService userService = new UserService();
        userService.login();
    }
}

public class UserService {

    public void login() {
        UserDao userService = new UserDao();
        userDao.login();
    }
}
```

那么有人说我们可以使用cache,使用单例就能保证项目中只存在一份instance,那为啥这种方案也不行?

假如新增一个简单需求,统计每一个接口的耗时统计? 那么我们该如何coding?
```java
public class UserController {
    public void login() {
        long start = System.currentTimeMillis();
        UserService userService = new UserService();
        userService.login();
        long end = System.currentTimeMillis();
        System.out.println("time is " + (end - start));
    }
}
```
那统计一下每个dao,操作SQL的执行时间呢。我们得硬编码,去做各种业务逻辑。代码冗余度会很高,在扩展性和维护性会很低,因此学习IOC的思想,看看Spring的优秀设计,对于做项目是非常重要的一件事。

# 学习IOC

![img.png](_assets/createObject.png)

## 基础定义
BeanDefinition是Spring框架中的一个概念，它用于描述Spring容器中的Bean对象的定义信息。每个Bean都需要一个BeanDefinition来描述它的类型、属性、依赖关系等信息。
BeanDefinition包含了以下属性：
- Bean的类型：描述Bean的具体类型，可以是普通的Java类、接口、抽象类等。
- Bean的作用域：描述Bean的生命周期和可见范围，如singleton（单例）、prototype（原型）、request（请求）、session（会话）等。
- Bean的依赖关系：描述Bean与其他Bean之间的依赖关系，包括依赖的Bean名称、依赖的Bean类型等。
- 属性值：描述Bean的属性值，包括属性名称和属性值。
- 初始化方法和销毁方法：描述Bean的初始化和销毁方法。
- 其他配置信息：描述其他的配置信息，如是否懒加载、是否自动装配等。
- 通过BeanDefinition，Spring容器可以根据配置文件或注解等方式获取到Bean的定义信息，并根据这些信息创建、管理和销毁Bean对象。
```java
/**
 * 简介：BeanDefinition描述了bean的实例属性以及构造方法,具体细节可以参考具体的实现类
 * A BeanDefinition describes a bean instance, which has property values,
 * constructor argument values, and further information supplied by
 * concrete implementations.
 * 重点：BeanDefinition是一个基础接口,最主要的目的是允许BeanFactoryPostProcessor修改属性值
 * <p>This is just a minimal interface: The main intention is to allow a
 * and other bean metadata.
 */
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
    // Bean的作用域
    String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
    
    String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;
    String getScope();

    //Bean的依赖关系
    void setDependsOn(@Nullable String... dependsOn);
    
    @Nullable
    String[] getDependsOn();

    //bean配置信息
    ConstructorArgumentValues getConstructorArgumentValues();
    
    //是否自动装配
    boolean isAutowireCandidate();
}
```

## Class依赖关系

![img.png](_assets/BeanFactoryRegister.png)

## bean的创建

bean的创建说简单点,其实就一种方法(反射),两种实现(jdk动态代理,cglib动态代理),前置条件要理解反射的两种实现

### cglib动态代理
cglib动态代理是一个基于ASM（一个开源Java字节码操作和分析框架）实现的代码生成库，用于在运行时生成代理类。相比于JDK动态代理，CGLIB动态代理更加强大和灵活，可以代理那些没有实现接口的类。
```java
// 目标类
public class TargetClass {
    public void doSomething() {
        System.out.println("Doing something...");
    }
}

// 代理类
public class ProxyClass implements MethodInterceptor {
    private Object target;

    public Object getProxy(Object target) {
        this.target = target;
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(this.target.getClass());
        enhancer.setCallback(this);
        return enhancer.create();
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("Before method invocation");
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("After method invocation");
        return result;
    }
}

public class Main {
    public static void main(String[] args) {
        ProxyClass proxyClass = new ProxyClass();
        TargetClass targetClass = (TargetClass) proxyClass.getProxy(new TargetClass());
        targetClass.doSomething();
    }
}
```
### jdk动态代理
jdk动态代理允许在运行时创建代理对象，可以在不修改原始对象的情况下，通过代理对象进行额外的操作或增加功能。
```java
public interface UserService {
    void addUser(String username);
    void deleteUser(String username);
}
public class UserServiceImpl implements UserService {
    @Override
    public void addUser(String username) {
        System.out.println("Add user: " + username);
    }

    @Override
    public void deleteUser(String username) {
        System.out.println("Delete user: " + username);
    }
}

public class UserServiceProxy implements InvocationHandler {
    private Object target;

    public UserServiceProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before invoking method: " + method.getName());
        Object result = method.invoke(target, args);
        System.out.println("After invoking method: " + method.getName());
        return result;
    }
}
//例子使用
public class Main {
    public static void main(String[] args) {
        UserService userService = new UserServiceImpl();
        UserService proxy = (UserService) Proxy.newProxyInstance(
                userService.getClass().getClassLoader(),
                userService.getClass().getInterfaces(),
                new UserServiceProxy(userService));

        proxy.addUser("Alice");
        proxy.deleteUser("Bob");
    }
}
```

## bean属性填充

代理的Object已创建,但具体的属性还没有填充。

AbstractAutowireCapableBeanFactory->createBean->doGetBean->populateBean->applyPropertyValues
```shell
		try{
                bw.setPropertyValues(new MutablePropertyValues(deepCopy));
                }
          catch(BeansException ex){
              //省略
        }
```

bean已经创建好,属性已经填充完毕,但如何将bean注册到springFactory还没有看到

## bean注册spring容器中

读取资源->装载->注册

XmlBeanDefinitionReader->loadBeanDefinitions->registerBeanDefinitions

DefaultBeanDefinitionDocumentReader->doRegisterBeanDefinitions

```java
public class DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader {
    protected void doRegisterBeanDefinitions(Element root) {
        //省略
        parseBeanDefinitions(root, this.delegate);
        //省略
    }

    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        if (delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();
            for (int i = 0; i < nl.getLength(); i++) {
                Node node = nl.item(i);
                if (node instanceof Element) {
                    Element ele = (Element) node;
                    //Bean处理,其中包含初始化,实例化
                    if (delegate.isDefaultNamespace(ele)) {
                        parseDefaultElement(ele, delegate);
                    }
                    else {
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        }
        else {
            delegate.parseCustomElement(root);
        }
    }

    private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        //省略
        if(xxx){
            
        }
        else if (delegate.nodeNameEquals(ele, "bean")) {
            processBeanDefinition(ele, delegate);
        }
        else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
            // 递归调用 省略
        
        }
    }

	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	    //将Bean放到Spring容器中
        // Register the final decorated instance.
		BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
		
		//默认是空实现,依赖是Spring事件机制实现 挖个坑,后面填
		getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
	}
}
```


## bean的扩展

既然提到了IOC的思想特性是解耦,那我们需要掌握Spring的Bean的生命周期,才能知道如何对Spring进行扩展
![img.png](_assets/beanLifeCycle.png)

### BeanFactoryPostProcessor

Spring的BeanFactoryPostProcessor接口是一个扩展点，它允许在Spring容器实例化和配置Bean之前对BeanDefinition进行修改。它的主要作用是对BeanDefinition进行定制化的处理，例如修改Bean的属性值、添加额外的Bean定义、移除不需要的Bean等。

BeanFactoryPostProcessor在Spring容器启动阶段执行，并且在Bean实例化之前执行。它可以对应用程序上下文中的所有BeanDefinition进行操作，包括自定义BeanDefinition和Spring内部创建的BeanDefinition。

通过实现BeanFactoryPostProcessor接口，可以在Spring容器启动时动态地修改BeanDefinition，从而实现对Bean的定制化处理。这种灵活性使得我们可以根据实际需求对Bean进行动态调整和配置，以满足特定的业务逻辑要求。

```java
@FunctionalInterface
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}

```

### BeanPostProcessor

Spring的BeanPostProcessor是一个特殊的接口，它允许在Spring容器实例化Bean和将其配置为应用程序组件之前和之后对它们进行自定义处理。

BeanPostProcessor接口定义了两个用于处理Bean的方法：

postProcessBeforeInitialization(Object bean, String beanName)：在初始化之前对Bean进行自定义处理。可以在此方法中修改Bean的属性值或执行一些自定义逻辑。

postProcessAfterInitialization(Object bean, String beanName)：在初始化之后对Bean进行自定义处理。可以在此方法中执行一些额外的初始化操作，或者对Bean进行包装。

BeanPostProcessor接口的实现类可以注册到Spring容器中，它们将自动应用于所有的Bean。可以使用BeanPostProcessor来实现一些通用的逻辑，例如日志记录、性能监控、事务管理等。

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

### InitializingBean

InitializingBean 是 Spring 框架中的一个接口，它定义了一个用于初始化 bean 的方法。
当一个 bean 实现了 InitializingBean 接口并且被 Spring 容器创建时，容器会调用 afterPropertiesSet() 方法来完成 bean 的初始化工作。

InitializingBean 接口的作用是允许 bean 在完成属性注入后执行一些自定义的初始化逻辑。通常情况下，我们可以在 afterPropertiesSet() 方法中实现一些需要在 bean 初始化阶段进行的操作，比如数据源的初始化、资源的加载等等。

使用 InitializingBean 接口的优点是，它提供了一个标准化的初始化方法，可以让开发人员在 bean 初始化阶段进行一些通用的操作，而不需要显式地在代码中调用初始化方法。'
另外，InitializingBean 接口还可以与其他 Spring 初始化相关的特性（例如 @PostConstruct 注解）一起使用，从而提供更大的灵活性。

```java
public interface InitializingBean {
  
    void afterPropertiesSet() throws Exception;

}
```
核心实现 AbstractAutowireCapableBeanFactory(类)->initializeBean(方法)->invokeInitMethods(方法), 
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory{
    //省略其余代码
protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd)
			throws Throwable {

		boolean isInitializingBean = (bean instanceof InitializingBean);
        
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isTraceEnabled()) {
				logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
						((InitializingBean) bean).afterPropertiesSet();
						return null;
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		if (mbd != null && bean.getClass() != NullBean.class) {
			String initMethodName = mbd.getInitMethodName();
			if (StringUtils.hasLength(initMethodName) &&
					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
}
```


### DisposableBean

DisposableBean是Spring框架提供的一个接口，用于在Spring容器关闭时执行一些清理操作。当一个Bean实现了DisposableBean接口，Spring容器在销毁该Bean时会自动调用其destroy()方法。

DisposableBean接口只定义了一个方法：

void destroy() throws Exception;

该方法允许Bean在销毁之前进行一些必要的清理工作，例如释放资源、关闭连接、取消注册等。实现该接口的Bean可以在destroy()方法中编写自己的清理逻辑。

需要注意的是，由于DisposableBean是Spring框架提供的接口，因此使用该接口会使你的代码与Spring框架产生一定的依赖关系。如果你不希望与Spring框架耦合，可以考虑使用其他方式来实现Bean的销毁逻辑，例如使用@PreDestroy注解或实现自定义的销毁方法。

```java
public interface DisposableBean {
    
    void destroy() throws Exception;

}
```

IOC的设计思想,针对于后端开发人员,我们需要关注的
第一个重点也就是Bean的生命周期,自定义Bean扩展,进行更高维度的抽象,例如spring的事务机制实现
第二个重点事件机制(ApplicationListener),熟悉它就能了解Spring与Nacos,Dubbo中间件集成是如何互相通信的

留坑待填









