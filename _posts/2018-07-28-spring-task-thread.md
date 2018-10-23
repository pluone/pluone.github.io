---
layout: post
title:  "Spring定时任务单线程执行之源码分析"
date:   2018-07-28 17:00:00 +0800
categories: 后端开发
---

spring定时任务`@EnableScheduling`默认单线程执行造成了的后果:  
假设有两个任务A,B,任务预期开始执行的时间分别是t(a),t(b),它们执行需要花费的时间分别为run_time(a),run_time(b)

1. 如果t(a)=t(b),那么单线程环境下只能有一个任务开始先执行,另一个任务必须等待前一个任务执行结束才会开始,假设A任务首先开始,那么B任务实际开始执行的时间是t(a)+run_time(a)
2. 如果t(a)<t(b),且run_time(a)>t(b)-t(a),那么单线程情况下b任务实际开始执行的时间时为t(a)+run_time(a)所以实际开发过程中如果有一个任务执行时间超长,其它的任务就都会一直被延迟.

### 我们来查找一下源代码,确认一下为何定时任务默认是单线程执行的:

1.  @EnableScheduling注解的源码上有一行`@Import(SchedulingConfiguration.class)`,导入了SchedulingConfiguration.class,在这个配置类中定义了ScheduledAnnotationBeanPostProcessor这个bean是专门用来处理@Scheduled注解的一个后置处理器,在这个类中实现了`org.springframework.beans.factory.config.BeanPostProcessor#postProcessAfterInitialization`这个方法,这个方法中扫描了对应类中的@Scheduled注解,然后开始调用`org.springframework.scheduling.annotation.ScheduledAnnotationBeanPostProcessor#processScheduled`这个方法来处理该注解,这个方法的主要目的将定时任务注册到ScheduledTaskRegistrar中,定时任务又分为三类fixDelay,fixRate,corn,之后这个bean后置处理器的工作就完成了.

2.  之后开始执行`org.springframework.scheduling.annotation.ScheduledAnnotationBeanPostProcessor#afterSingletonsInstantiated`这是一个生命周期方法,这个生命周期方法又调用了`org.springframework.scheduling.annotation.ScheduledAnnotationBeanPostProcessor#finishRegistration`,又调用了`org.springframework.scheduling.config.ScheduledTaskRegistrar#afterPropertiesSet`,又调用了`org.springframework.scheduling.config.ScheduledTaskRegistrar#scheduleTasks`这里才是重头戏,前面都是在注册定时任务,这里将要启动定时任务,通过`Executors.newSingleThreadScheduledExecutor();`启动了一个单线程的taskScheduler,代码段摘录如下:

```java
protected void scheduleTasks() {
    long now = System.currentTimeMillis();

    if (this.taskScheduler == null) {
        this.localExecutor = Executors.newSingleThreadScheduledExecutor();
        //这里设置了一个单线程的taskScheduler
        this.taskScheduler = new ConcurrentTaskScheduler(this.localExecutor);
    }
    //之后的代码的主要任务就是往scheduledFutures这个set中添加任务,开始执行
    if (this.triggerTasks != null) {
        for (TriggerTask task : this.triggerTasks) {
            this.scheduledFutures.add(this.taskScheduler.schedule(
                    task.getRunnable(), task.getTrigger()));
        }
    }
    ......
}
```
所以我们可以通过定义一个配置类来启用多线程,如下:

```java
@Configuration
public class TaskSchedulerConfig {
    @Bean
    public TaskScheduler taskScheduler() {
        //定时任务的线程池设置为10个，默认是单线程
        return new ConcurrentTaskScheduler(Executors.newScheduledThreadPool(10));
    }
}
```