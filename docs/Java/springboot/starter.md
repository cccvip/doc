##  学会理解,并使用
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
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    //省略
}
```

###  自动配置
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









