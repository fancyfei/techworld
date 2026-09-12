# Spring Task 核心架构

> SpringBoot中，Task 驱动入口是`@EnableScheduling`。默认的调度器是 `ThreadPoolTaskScheduler`，底层是对 JDK `ScheduledThreadPoolExecutor` 的封装。同时负责线程生命周期、任务调度以及 `TaskScheduler` 接口适配。
>
> 下面示例基于 Spring Boot 3.x（Java 17+）。

## 1 入口链路

Spring Task 的核心运作逻辑可以总结为：**注解扫描与解析** --> **任务暂存与注册** --> **调度与线程池执行**。

```
@EnableScheduling
       │
       ▼
ScheduledAnnotationBeanPostProcessor (Bean后置处理器)
       │ (扫描所有 Bean 的 @Scheduled 方法)
       ▼
Registrar (ScheduledTaskRegistrar)
       │ (匹配策略并组装 Task)
       ▼
TaskScheduler (默认的 ThreadPoolTaskScheduler)
       │
       ▼
ScheduledExecutorService (JDK 线程池提交并调度)
```

### 1.1 加载与解析阶段

1. **触发入口**：`@EnableScheduling` 导入了 `SchedulingConfiguration` 配置类，注册了一个关键 Bean —— **`ScheduledAnnotationBeanPostProcessor`**。
2. **扫描与注册**：该 BeanPostProcessor 在 Bean 初始化后（`postProcessAfterInitialization`），扫描 `@Scheduled` 方法。
3. **任务封装**：针对不同的注解参数，封装为不同的 `Task` 对象：
   - `fixedRate` / `fixedDelay` -> `IntervalTask`
   - `cron` -> `CronTask`（内部通过 `CronExpression` 进行下次时间解析）
4. **提交 Registrar**：封装好的 Task 会被添加进 `ScheduledTaskRegistrar` 管理器中。

### 1.2 任务调度与线程池执行

1. **线程池初始化**：当所有单例 Bean 初始化完成后，Spring 会触发 `ScheduledTaskRegistrar` 的初始化（`afterPropertiesSet`）。
2. **查找 TaskScheduler**：
   - 寻找类型为 `TaskScheduler` 或 `ScheduledExecutorService` 的 Bean。
   - **注意**：如果找不到，默认会创建一个**单线程**的 `ThreadPoolTaskScheduler`（或 `Executors.newSingleThreadScheduledExecutor()`）。
3. **注册到底层线程池**：`TaskScheduler` 将 Task 转化提交给底层的 `ScheduledExecutorService` 执行：
   - `fixedRate` -> `scheduleAtFixedRate()`
   - `fixedDelay` -> `scheduleWithFixedDelay()`
   - `cron` -> 动态计算下一次执行时间，使用 `schedule(runnable, delay)` 循环触发。

## 2 TaskScheduler 接口体系

### 2.1 顶层接口定义

```java
public interface TaskScheduler {
    // 1. 一次性/ Cron/ Trigger 任务
    ScheduledFuture<?> schedule(Runnable task, Trigger trigger);
    ScheduledFuture<?> schedule(Runnable task, Instant startTime);

    // 2. 固定频率任务（fixedRate 语义）
    ScheduledFuture<?> scheduleAtFixedRate(Runnable task, Duration period);

    // 3. 固定间隔任务（fixedDelay 语义）
    ScheduledFuture<?> scheduleWithFixedDelay(Runnable task, Duration period);
}
```

`Trigger` 抽象用于动态计算下一次执行时间：

```java
public interface Trigger {
    Instant nextExecution(TriggerContext triggerContext);
}

public interface TriggerContext {
    Instant lastScheduledExecution(); // 上次计划触发时间
    Instant lastActualExecution();    // 上次实际开始时间
    Instant lastCompletion();         // 上次实际结束时间
}
```

### 2.2 调度器实现

默认的调度器实现为 `newSingleThreadScheduledExecutor`，底层是对 JDK `ScheduledThreadPoolExecutor` 的封装。

当容器中未显式配置 `TaskScheduler` 或 `ScheduledExecutorService` 的 Bean 时，`ScheduledTaskRegistrar` 触发兜底策略：

```java
// ScheduledTaskRegistrar 内部逻辑（简化）
protected void scheduleTasks() {
    if (this.taskScheduler == null) {
        // 兜底：未配置时创建单线程调度器
        this.localExecutor = Executors.newSingleThreadScheduledExecutor();
        this.taskScheduler = new ConcurrentTaskScheduler(this.localExecutor);
    }
    // 遍历注册任务
    for (CronTask task : this.cronTaskList) {
        this.taskScheduler.schedule(task.getRunnable(), task.getTrigger());
    }
    // ...注册 fixedRate / fixedDelay 任务
}
```

## 3 Cron 任务与循环 Reschedule 模型

`fixedRate` 与 `fixedDelay` 直接依赖 JDK 线程池的周期调度。Cron 任务由于触发时间间隔不固定，无法直接映射为 JDK 的固定周期 API，Spring 通过 **循环 Reschedule（重新调度）机制** 实现。

### 3.1 `ReschedulingRunnable` 执行流程

所有 Trigger 任务最终被封装为 `ReschedulingRunnable` 提交给执行器（伪代码）：

```java
public class ReschedulingRunnable extends DelegatingErrorHandlingRunnable implements ScheduledFuture<Object> {
    private final Trigger trigger;
    private final SimpleTriggerContext triggerContext = new SimpleTriggerContext();

    @Override
    public void run() {
        Date actualExecutionTime = new Date();
        // 1. 执行目标任务
        super.run();
        Date completionTime = new Date();
        
        // 2. 更新上下文并计算下一次触发时间
        this.triggerContext.update(this.scheduledExecutionTime, actualExecutionTime, completionTime);
        Date nextExecutionTime = this.trigger.nextExecutionTime(this.triggerContext);
        
        // 3. 重新向调度器提交下一次任务
        if (nextExecutionTime != null) {
            this.currentFuture = this.executor.schedule(this, nextExecutionTime);
        }
    }
}
```

### 3.2 Cron 调度的核心设计

1. **基准时间计算**：`CronTrigger` 默认以 `lastCompletionTime`（上次完成时间）为基准计算下一次触发点。若任务执行时间过长导致错过中间的触发点，不会补偿补跑，直接顺延至下一个满足表达式的时间。
2. **调度独立性**：每次执行完成后才重新计算并调用 `executor.schedule()`。长任务会跳过错过的点顺延到下一个。
3. **可控中断**：若 `Trigger.nextExecutionTime()` 返回 `null`，循环调度终止。