本来想聊聊定时器相关的内容,但是太多,特意拆分一下成每个小块,其中有些会涉及到源码的解读。

## 一般是怎么认识Scheduled定时器
> 单体项目中,定时轮询任务。修改表数据,触发下游业务操作等等

## 怎么使用
> springboot体系中,在启动类上加入@EnableScheduling注解。在任务方法上加入 @Scheduled注解。

```java
    @Scheduled(fixedRate = 2000)
    public void printLog(){
        log.info("定时打印");
    }
```

问题开始了,一个定时器这么写没问题,如果有10个定时器,并且crontab表达式都是一样,那底层实现是如何? 是并行执行,还是串行执行?

压测代码如下
```java
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
```
结果如图所示
[sleep](_assets/sleep.png) 
[sleep](_assets/sleep2.png) 
这个结果论证了一件事,默认是串行执行的。因为大量的定时的时间都是固定每2000ms执行一次,但因为各个方法休眠的时长不一样。

瞄一下源码是怎么处理的
1 定位Scheduled注解的后置处理器ScheduledAnnotationBeanPostProcessor
看到一个定时器任务相关的类
```java
	private final Map<Object, Set<ScheduledTask>> scheduledTasks = new IdentityHashMap<>(16);
```
2 查看postProcessAfterInitialization方法
源码如下
```java
            // Non-empty set of methods
			annotatedMethods.forEach((method, scheduledMethods) ->
						scheduledMethods.forEach(scheduled -> processScheduled(scheduled, method, bean)));
                if (logger.isTraceEnabled()) {
					logger.trace(annotatedMethods.size() + " @Scheduled methods processed on bean '" + beanName +
							"': " + annotatedMethods);
				}
```
processScheduled是处理定时器相关类,简单理解就是创建定时器相关Runnable
最后找到这一行代码,将任务注册到 ScheduledTaskRegistrar执行
```java
// Finally register the scheduled tasks
			synchronized (this.scheduledTasks) {
				Set<ScheduledTask> regTasks = this.scheduledTasks.computeIfAbsent(bean, key -> new LinkedHashSet<>(4));
				regTasks.addAll(tasks);
			}
```

3 查看ScheduledTaskRegistrar做了什么操作
几大定时器的属性
```java
	@Nullable
	private List<TriggerTask> triggerTasks;

	@Nullable
	private List<CronTask> cronTasks;

	@Nullable
	private List<IntervalTask> fixedRateTasks;

	@Nullable
	private List<IntervalTask> fixedDelayTasks;

```
最终执行类
```java
	protected void scheduleTasks() {
		if (this.taskScheduler == null) {
			this.localExecutor = Executors.newSingleThreadScheduledExecutor();
			this.taskScheduler = new ConcurrentTaskScheduler(this.localExecutor);
		}
		if (this.triggerTasks != null) {
			for (TriggerTask task : this.triggerTasks) {
				addScheduledTask(scheduleTriggerTask(task));
			}
		}
		if (this.cronTasks != null) {
			for (CronTask task : this.cronTasks) {
				addScheduledTask(scheduleCronTask(task));
			}
		}
		if (this.fixedRateTasks != null) {
			for (IntervalTask task : this.fixedRateTasks) {
				addScheduledTask(scheduleFixedRateTask(task));
			}
		}
		if (this.fixedDelayTasks != null) {
			for (IntervalTask task : this.fixedDelayTasks) {
				addScheduledTask(scheduleFixedDelayTask(task));
			}
		}
	}
```

```java
	@Nullable
	public ScheduledTask scheduleCronTask(CronTask task) {
		ScheduledTask scheduledTask = this.unresolvedTasks.remove(task);
		boolean newTask = false;
		if (scheduledTask == null) {
			scheduledTask = new ScheduledTask(task);
			newTask = true;
		}
		if (this.taskScheduler != null) {
			scheduledTask.future = this.taskScheduler.schedule(task.getRunnable(), task.getTrigger());
		}
		else {
			addCronTask(task);
			this.unresolvedTasks.put(task, scheduledTask);
		}
		return (newTask ? scheduledTask : null);
	}
```
到这差不多就可以结束,再往下看嵌套的代码就太多了。

## 整理一下
crontab的定时器,会按照bean顺序添加到默认的线程队列中执行,因此是串行执行的。








