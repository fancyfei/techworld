# 单机定时任务调度框架 Spring Task

> Spring Task 是 Spring 的轻量级定时任务框架。@Scheduled是核心注解。
>
> 下面的例子基于 Spring Boot 3.x（Java 17+）。

## 1 基本用法

两个注解：

- `@EnableScheduling`：标注在配置类上，开启定时任务；
- `@Scheduled`：标注在任务方法上，声明触发策略，方法所在类必须注册为 Spring Bean。

### 1.1 @Scheduled注解
```java
@SpringBootApplication
@EnableScheduling
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

@Component
public class ReportTask {

    private final Logger log = LoggerFactory.getLogger(ReportTask.class);

    @Scheduled(fixedRate = 10_000)   // 每 10 秒执行一次
    public void generateReport() {
        log.info("Job，线程：{}", Thread.currentThread().getName());
    }
}
```

### 1.2 编程式注册：SchedulingConfigurer

注解方式的任务在编码期就固定了。触发策略要从数据库或配置中心动态读取、任务列表运行期可变时，改用 `SchedulingConfigurer` 编程式注册：
```java
@Configuration
@EnableScheduling
public class DynamicSchedulingConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        // Trigger 任务：Runnable task, Trigger trigger
        registrar.addTriggerTask(this::runTaskFunc,
            triggerContext -> new CronTrigger("0/10 * * * * ?")
                    .nextExecution(triggerContext));
        // 其他的addXxxTask还有cron 任务、固定间隔任务、固定频率任务
    }
}
```

与注解方式的区别：

- `addTriggerTask` 的 `Trigger` 回调在每次执行后都会被调用来计算下次触发时间，运行期可以修改 cron 下一次重新计算触发时间；
- 需要运行期动态增删任务时，用 `registrar.scheduleCronTask(...)` 系列方法（返回 `ScheduledTask`，持有引用即可 `cancel()` 取消），`addXxxTask` 注册的任务无法在运行期单独摘除；
- 同一个 `registrar` 上可同时设置线程池， `registrar.setTaskScheduler(...)`。


## 2 触发方式

`@Scheduled` 主要提供三种周期触发方式：fixedRate、fixedDelay 和 cron。此外还可以通过 initialDelay 控制首次执行时间，并通过 timeUnit、zone、scheduler 等属性进一步控制调度行为。

### fixedRate 与 fixedDelay

都接受毫秒值，区别在于时间基准：

| 参数                | 时间基准     | 上次耗时 8s、间隔 5s 时      |
| ------------------- | ------------ | ---------------------------- |
| `fixedRate = 5000`  | 上次开始时间 | 当前任务结束后尽快执行下一次 |
| `fixedDelay = 5000` | 上次结束时间 | 再等 5 秒执行                |

fixedRate 适合要求固定节奏的场景（定时刷新缓存）；fixedDelay 适合任务间不允许重叠的场景（轮询外部接口）。

### cron 表达式

日历式调度，6 位域：

```
秒 分 时 日 月 周
```

```java
@Scheduled(cron = "0 0 2 * * ?")          // 每天凌晨 2 点
@Scheduled(cron = "0 */5 * * * ?")        // 每 5 分钟
@Scheduled(cron = "0 0 9-18 * * MON-FRI") // 工作日 9~18 点整点
```

常用特殊字符：`*` 任意值、`?` 不指定（日/周域）、`/` 步进、`L` 最后一天。

### 触发参数外部化

`fixedRateString`、`fixedDelayString`、`cron` 的字符串形式支持 `${...}` 占位符，把触发策略放到 `application.yml` 里按环境调整：

```java
@Scheduled(fixedDelayString = "${task.sync.interval:60000}")
public void syncData() { ... }
```

### initialDelay 与 zone

`initialDelay` 控制首次执行的延迟，避免应用启动阶段就触发；`zone` 为 cron 指定时区，仅跨时区部署时需要：

```java
@Scheduled(initialDelay = 30_000, fixedDelay = 60_000)  // 启动 30 秒后开始，之后每 60 秒
@Scheduled(cron = "0 0 2 * * ?", zone = "Asia/Shanghai")
```

## 3 使用约束

1. 方法必须无参，返回值被忽略。
2. 方法所在对象需要由 Spring 容器管理。
3. `fixedRate` 不是精确频率。受调度线程竞争和任务耗时影响，实际触发时间存在抖动；有精度要求的场景需配置独立线程池（见 1.4）或使用专用调度系统。
4. 同一个 @Scheduled 方法可以通过可重复注解配置多个调度规则，但多个规则可能并发或连续触发。

## 4 单线程阻塞

Spring 默认调度器只有 1 个线程，所有 `@Scheduled` 任务共享。一个任务耗时长，其余任务全部排队，触发时间持续顺延——生产上"定时任务不准时"。三种处理手段：

### 4.1. 配置调度线程池

实现 `SchedulingConfigurer` 注入 `ThreadPoolTaskScheduler`：

```java
@Configuration
@EnableScheduling
public class SchedulingConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(8);            // 按同时运行的任务数评估
        scheduler.setThreadNamePrefix("job-");
        scheduler.initialize();
        registrar.setTaskScheduler(scheduler);
    }
}
```

Spring Boot 下不需要深度定制时，直接用配置项：

```yaml
spring:
  task:
    scheduling:
      pool:
        size: 8
      thread-name-prefix: job-
```

### 4.2. 长任务交给 @Async

内部业务耗时长时，用 `@Async` 把它转移到独立业务线程池，调度线程立即释放：

```java
@Async("bizExecutor")            // 独立业务线程池
@Scheduled(fixedDelay = 60_000)
public void heavyTask() {
    // 耗时操作
}
```

### 4.3. 拆分任务粒度

任务太大，可以分片分步设计，单次执行只处理一小批数据。

## 5 异常处理

### 默认行为

任务抛出的未捕获异常会被调度框架捕获并记录日志，不中断后续调度。默认日志只有一条框架级的 `Unexpected error occurred in scheduled task`。

### 统一 ErrorHandler

配置调度器时设置 `ErrorHandler`，集中记录异常：

```java
ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
scheduler.setErrorHandler(t -> {
    log.error("定时任务执行异常", t);
    // 接入告警
});
```

`TaskUtils` 提供两个内置处理器：默认的 `LOG_AND_SUPPRESS_ERROR_HANDLER`（记日志并吞掉异常）和 `LOG_AND_PROPAGATE_ERROR_HANDLER`（记日志后向上抛出）。

### 任务内 try-catch

`ErrorHandler` 只兜底任务抛出去的异常。内部业务需要可以自己try-catch：

```java
@Scheduled(cron = "0 0 2 * * ?")
public void dailyJob() {
    for (TaskStep step : steps) {
        try {
            step.execute();
        } catch (Exception e) {
            log.error("步骤 {} 执行失败，继续后续步骤", e);
        }
    }
}
```

### @Async 任务的异常

`@Async` 任务的异常发生在独立线程池中，`ErrorHandler` 和任务外层 try-catch 都不起作用，需实现 `AsyncConfigurer` 的 `AsyncUncaughtExceptionHandler`：

```java
@Configuration
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("异步任务异常，方法：{}", method.getName(), ex);
        };
    }
}
```

Spring 6.1 开始，`@Scheduled` 还支持返回 Reactive `Publisher` 的方法，其异常处理机制有所不同。

## 6 集群防重

Spring Task 是单机调度，多实例部署时每个实例都会触发同一任务，造成重复执行。两种处理思路：

- **分布式锁**：接入 ShedLock，执行前抢锁，持锁期间其他实例自动跳过，接入成本最低；也可基于 Redis `SETNX` 实现一个。锁的过期时间必须大于任务最大可能耗时。
- **分布式调度平台**：任务量大、需要分片并行或统一管控时，引入 XXL-JOB / ElasticJob，由调度中心保证任务只在一台实例上执行。

## 7 与其他定时方案对比

| 方案                       | 特点                                         | 适用场景                             |
| -------------------------- | -------------------------------------------- | ------------------------------------ |
| `java.util.Timer`          | JDK 原生，单线程，异常导致整个 Timer 终止    | 已不推荐，仅存留于遗留代码           |
| `ScheduledExecutorService` | JDK 并发库，线程池化，无 Spring 生命周期管理 | 不需要 Spring 集成的简单场景         |
| Spring Task                | 声明式、与容器生命周期集成、配置简单         | 单机定时任务首选                     |
| Quartz                     | 持久化 JobStore、集群调度、misfire 策略完善  | 需要任务持久化或集群调度的重量级场景 |

集群方案直接引入分布式调度平台。