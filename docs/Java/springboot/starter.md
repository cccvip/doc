## 学会理解,并使用

> 自定义各种starter为业务服务

## 名词解释

Spring Boot Starter 是 Spring Boot 的一个重要特性，它用于简化依赖和项目配置，让我们能够更快速地开发 Spring 应用程序。

- 自动配置：Spring Boot Starter 自动配置是基于 Spring 4.0 的条件注解（@Conditional），在满足某种条件时，Spring 容器就会自动给我们配置 Bean。
- Starter POM：Starter 是一个 Maven 项目对象模型（POM），它包含了项目的所有依赖信息。当我们在项目中引入 Starter 时，这些依赖就会自动被添加进项目中。
- Spring.factories 文件：Spring Boot 在启动时会自动扫描所有的 Spring.factories 文件，并加载其中的配置。这个文件位于项目的 resources/META-INF 目录下

## 原理分析

#### 举个例子

SpringBoot开启web功能,只需要引入该依赖即可

```shell
  <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  //增加启动主函数
  
  @SpringBootApplication
  public class ApiApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiApplication.class, args);
    }
}
```

@SpringBootApplication注解是一个组合注解,里面包含各种注解。使用@EnableAutoConfiguration默认开启自动装配

```java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)})
public @interface SpringBootApplication {
    //省略
}
```

### 自动配置

(以spring-boot-starter-web为例)

#### 名词解释

配置也就是配置文件,常见application.yml中,例如

```yaml
server:
  port: 8081
```

自动也就意味着,默认开启此功能,就能将JAVA服务,摇身一变、成一个web服务器,并且能提供Http请求的接口服务器

### @EnableAutoConfiguration注解分析

@EnableAutoConfiguration同样也是一个组合注解

```java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    //省略
}
```

重点关注@Import引入的AutoConfigurationImportSelector

#### @Import注解

根据官方对Import的注解描述,一共有四种实现方式

- @Configuration
- 实现ImportSelector的接口
- 实现ImportBeanDefinitionRegistrar接口
- 使用普通的@Componet注解

```java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {
    /**
     * {@link Configuration @Configuration}, 
     * {@link ImportSelector},
     * {@link ImportBeanDefinitionRegistrar}, 
     * or regular component classes to import.
     */
    Class<?>[] value();
}
```

Spring-Boot实现的方式是第二种,通过扫描外部的文件,被扫描到Spring中

```java
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
        ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        }
        //获取配置属性
        AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
        return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
    }

    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        //加载外部文件 
        // META-INF/spring.factories
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
                getBeanClassLoader());
        Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
                + "are using a custom packaging, make sure that file is correct.");
        return configurations;
    }
}
```

#### spring.factories

```shell
org.springframework.boot.autoconfigure.EnableAutoConfiguration=XXX配置类
```

那究竟是在Spring的哪一个生命周期处理的呢? 如何解析spring.factories,并且将实现类注册到Spring中?
> 脑子过一下Spring的执行周期,只能在BeanFactoryProcessor身上做文章,将spring.factories,生成为BeanDefinition。

**Spring处理逻辑,伪代码如下**

- AbstractApplicationContext-refresh(func)-> 执行invokeBeanFactoryPostProcessors(beanFactory);

```shell
public static void invokeBeanFactoryPostProcessors(
        ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
      if (beanFactory instanceof BeanDefinitionRegistry){
        //省略
        for(BeanFactoryPostProcessor postProcessor:beanFactoryPostProcessors){
           if(postProcessor instanceof BeanDefinitionRegistryPostProcessor){
 
            BeanDefinitionRegistryPostProcessor registryProcessor= (BeanDefinitionRegistryPostProcessor)postProcessor;
            //外部扫描生成BeanDefinition
            registryProcessor.postProcessBeanDefinitionRegistry(registry);
            
            registryProcessors.add(registryProcessor);
           }
        else{
            regularPostProcessors.add(postProcessor);
          }
          }
        }
        //省略
    }
```

- ConfigurationClassPostProcessor->postProcessBeanDefinitionRegistry(方法)

```shell
    
        // Parse each @Configuration class
		ConfigurationClassParser parser = new ConfigurationClassParser(
		this.metadataReaderFactory, this.problemReporter, this.environment,
		this.resourceLoader, this.componentScanBeanNameGenerator, registry);
		do{
		  //例如Spring的指定package下的 
		  parser.parse(candidates);
		  

	}while()
	
	
```

- ConfigurationClassParser->doProcessConfigurationClass解析

```shell
	  // Process any @Import annotations
	  processImports(configClass, sourceClass, getImports(sourceClass), filter, true);
    
```

递归扫描@Configuration

- 回到上一步parser.parse(candidates)
  获取外部所有@Configuration的class,然后执行 this.reader.loadBeanDefinitions(configClasses);

```shell
	public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
		TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
		for (ConfigurationClass configClass : configurationModel) {
			loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
		}
	}
	  
	private void loadBeanDefinitionsForConfigurationClass(
			ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
        //将外部引入的class生成为BeanDefinition,加入注册表中
		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
	}
	
```  

- 后续加载就按照Spring的正常流程处理,进行Bean的初始化,属性填充,实例化(前置处理,后置处理)等等流程









