# 常见注解

为了更好的理解Spring的IOC与AOP特性,而不只是浮在表面。下面将介绍一些常见注解,以及如何debug代码,明白注解的实现。

举一个例子

```java
import javax.annotation.Resource;

@RestController
@RequestMapping(value = "app")
public class AppController {

    @Resource
    AppService appService;

    @RequestMapping(value = "/ping")
    public String ping() {
        return "pong";
    }

}

```

## @RestController

@RestController注解是一个特殊的@Controller，它包含了@ResponseBody注解。这意味着我们无需在每个方法上都添加@ResponseBody注解，因为当类使用@RestController注解时，就默认为所有方法都使用了@ResponseBody。
所以简单来说，@RestController是帮助我们更方便的创建RESTful Web服务的一种快捷方式

### how

@RestController由@Controller跟@ResponseBody,两个注解组成,

- @ResponseBody是标识当前请求是以restful风格返回JSON数据。
- @Controller 从继承 @Component,标识当前Bean需要被扫描,需要被Spring管理

```java

@Controller
@ResponseBody
public @interface RestController {

    @AliasFor(annotation = Controller.class)
    String value() default "";
}

@Component
public @interface Controller {
    @AliasFor(annotation = Component.class)
    String value() default "";
}

```

## @RequestMapping

RequestMapping才是SpringMVC架构的核心












