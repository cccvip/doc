## 一般是怎么认识Scheduled定时器
> 单体项目中,定时轮询任务。修改表数据,触发下游业务操作等等

## how
> springboot体系中,在启动类上加入@EnableScheduling注解。在任务方法上加入 @Scheduled注解。

```java
@Component
public class AppSchedule{
    @Scheduled(fixedRate = 2000)
    public void printLog(){
        log.info("定时打印");
    }
}
```

## 问题一
如果有10个定时器,并且crontab表达式都是一样,那底层实现是如何? 是并行执行,还是串行执行?

压测代码如下
```java
@Component
public class AppSchedule {
    @Scheduled(fixedRate = 2000)
    public void printLog1() {
        try {
            Thread.sleep(1000 * 1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info("定时打印1");
    }

    @Scheduled(fixedRate = 2000)
    public void printLog2() {
        try {
            Thread.sleep(1000 * 2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info("定时打印2");
    }

    @Scheduled(fixedRate = 2000)
    public void printLog3() {
        try {
            Thread.sleep(1000 * 3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info("定时打印3");
    }
}
```

这个结果论证了一件事,默认是串行执行的。因为大量的定时的时间都是固定每2000ms执行一次,但因为各个方法休眠的时长不一样。

## 如何实现
### 定位Scheduled注解的后置处理器ScheduledAnnotationBeanPostProcessor
看到一个定时器任务相关的类
```java
public class ScheduledAnnotationBeanPostProcessor implements ScheduledTaskHolder, 
        MergedBeanDefinitionPostProcessor, 
        DestructionAwareBeanPostProcessor, 
        Ordered, EmbeddedValueResolverAware, BeanNameAware, BeanFactoryAware, 
        ApplicationContextAware, 
        SmartInitializingSingleton, ApplicationListener<ContextRefreshedEvent>, DisposableBean {
    //省略代码
    private final Map<Object, Set<ScheduledTask>> scheduledTasks = new IdentityHashMap<>(16);
    
}
```
### 查看postProcessAfterInitialization方法
源码如下
```java
public class ScheduledAnnotationBeanPostProcessor {
    
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (!(bean instanceof AopInfrastructureBean) && !(bean instanceof TaskScheduler) && !(bean instanceof ScheduledExecutorService)) {
            Class<?> targetClass = AopProxyUtils.ultimateTargetClass(bean);
            if (!this.nonAnnotatedClasses.contains(targetClass) && AnnotationUtils.isCandidateClass(targetClass, Arrays.asList(Scheduled.class, Schedules.class))) {
                
                Map<Method, Set<Scheduled>> annotatedMethods = MethodIntrospector.selectMethods(targetClass, (method) -> {
                    Set<Scheduled> scheduledMethods = AnnotatedElementUtils.getMergedRepeatableAnnotations(method, Scheduled.class, Schedules.class);
                    return !scheduledMethods.isEmpty() ? scheduledMethods : null;
                });
                if (annotatedMethods.isEmpty()) {
                    this.nonAnnotatedClasses.add(targetClass);
                    if (this.logger.isTraceEnabled()) {
                        this.logger.trace("No @Scheduled annotations found on bean class: " + targetClass);
                    }
                } else {
                    //核心代码逻辑
                    annotatedMethods.forEach((method, scheduledMethods) -> {
                        scheduledMethods.forEach((scheduled) -> {
                            //执行方法    
                            this.processScheduled(scheduled, method, bean);
                        
                        });
                    });
                    
                    if (this.logger.isTraceEnabled()) {
                        this.logger.trace(annotatedMethods.size() + " @Scheduled methods processed on bean '" + beanName + "': " + annotatedMethods);
                    }
                }
            }

            return bean;
        } else {
            return bean;
        }
    }
}
```
processScheduled是处理定时器相关类,简单理解就是创建定时器相关Runnable,最后找到这一行代码,将任务注册到 ScheduledTaskRegistrar执行
```java
public class ScheduledAnnotationBeanPostProcessor{
    private void finishRegistration() {
        //将注解扫描的任务类注册到ScheduledTaskRegistrar
    }
}
```
## 总结
crontab的定时器,会按照bean顺序添加到默认的线程队列中执行,因此是串行执行的。从这里也可以看到Spring良好的扩展性








