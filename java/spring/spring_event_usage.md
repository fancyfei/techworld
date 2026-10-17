# Spring Event 使用指南

## 一、典型应用

### 1. 定义事件

从 Spring Framework 4.2 开始，事件可以是任意普通 Java 对象，不需要继承 `ApplicationEvent`。

```java
public record UserRegisteredEvent(Long userId, String email) {
}
```

事件对象只承载数据，不包含行为。使用 `record` 或 `final` 字段保证不可变性，避免监听器之间通过事件对象互相影响。

### 2. 发布事件

业务代码注入 `ApplicationEventPublisher`，调用 `publishEvent()` 提交事件对象。

```java
@Service
public class UserService {

    private final UserRepository userRepository;
    private final ApplicationEventPublisher eventPublisher;

    public UserService(UserRepository userRepository,
                       ApplicationEventPublisher eventPublisher) {
        this.userRepository = userRepository;
        this.eventPublisher = eventPublisher;
    }

    public void register(User user) {
        userRepository.save(user);
        eventPublisher.publishEvent(
            new UserRegisteredEvent(user.getId(), user.getEmail())
        );
    }
}
```

发布者只负责提交事件对象，不关心有哪些监听器。

### 3. 监听事件

在任意 Spring Bean 的方法上标注 `@EventListener`，方法参数类型即监听的事件类型。

```java
@Component
public class WelcomeEmailListener {

    @EventListener
    public void onUserRegistered(UserRegisteredEvent event) {
        // 发送欢迎邮件
    }
}
```

同一事件可以有任意多个监听器，新增处理逻辑只需要新增监听器 Bean。

## 二、常用用法

### 1. 条件过滤

`condition` 属性使用 SpEL 表达式过滤事件，条件不成立时方法不会被调用。

```java
@EventListener(
    condition = "#event.email().endsWith('@corp.example.com')"
)
public void onCorpUserRegistered(UserRegisteredEvent event) {
    // 仅处理企业邮箱注册用户
}
```

表达式中的 `#event` 来自方法参数名，依赖编译时保留参数名信息（`-parameters` 编译参数，Spring Boot 默认开启）。不依赖参数名的写法是 `#root.args[0]`。

条件判断由事件框架执行，发布者不需要感知。

### 2. 一个方法监听多种事件

`classes` 属性声明多个事件类型，方法可以不声明参数，仅作为触发器。

```java
@EventListener({UserRegisteredEvent.class, UserActivatedEvent.class})
public void onUserChanged() {
    // 刷新用户缓存
}
```

需要读取事件数据时，参数声明为这些类型的公共父类型。

### 3. 监听器顺序

多个同步监听器需要固定先后顺序时，使用 `@Order`，数值小的先执行。

```java
@EventListener
@Order(1)
public void checkBlacklist(UserRegisteredEvent event) {
}

@EventListener
@Order(2)
public void initPoints(UserRegisteredEvent event) {
}
```

`@Order` 只约束监听器被调用的先后。异步监听器在不同线程中执行，完成顺序不受 `@Order` 约束。

### 4. 返回值发布后续事件

监听方法返回非空对象时，该对象会作为新事件继续发布；返回集合则逐个发布。

```java
@EventListener
public UserAuditEvent onUserRegistered(UserRegisteredEvent event) {
    // 处理注册事件
    return new UserAuditEvent(event.userId(), "REGISTERED");
}
```

异步监听方法（`@Async`）不支持返回值再发布，需要在方法内注入 `ApplicationEventPublisher` 手动发布。

### 5. 泛型事件

监听器可以按泛型参数区分事件，例如只接收 `EntityCreatedEvent<Person>`：

```java
@EventListener
public void onPersonCreated(EntityCreatedEvent<Person> event) {
}
```

由于类型擦除，直接发布 `EntityCreatedEvent<Person>` 实例时泛型信息在运行期不可见，匹配不会生效，监听器静默收不到事件。两种解决方式：

- 发布具体子类：`class PersonCreatedEvent extends EntityCreatedEvent<Person> { }`。
- 让事件类实现 `ResolvableTypeProvider`，在运行期报告泛型类型：

```java
public class EntityCreatedEvent<T> implements ResolvableTypeProvider {

    private final T entity;

    public EntityCreatedEvent(T entity) {
        this.entity = entity;
    }

    @Override
    public ResolvableType getResolvableType() {
        return ResolvableType.forClassWithGenerics(
            getClass(), ResolvableType.forInstance(entity)
        );
    }
}
```

泛型事件匹配失败时不报错、不触发监听器，排查困难。实体类型较少的场景，优先使用具体事件类。

## 三、异步执行

### 1. 开启异步监听

启用异步支持，并在监听方法上标注 `@Async`。

```java
@Configuration
@EnableAsync
public class AsyncConfig {
}
@Async
@EventListener
public void sendWelcomeEmail(UserRegisteredEvent event) {
    // 在其他线程中执行
}
```

### 2. 配置线程池

`@Async` 默认使用容器中的执行器。生产环境应显式定义线程池，而不是依赖默认执行器。

```java
@Bean
public ThreadPoolTaskExecutor eventExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(8);
    executor.setQueueCapacity(1000);
    executor.setThreadNamePrefix("event-");
    executor.initialize();
    return executor;
}
@Async("eventExecutor")
@EventListener
public void sendWelcomeEmail(UserRegisteredEvent event) {
}
```

另一种方式是为广播器统一配置执行器：向容器注册名为 `applicationEventMulticaster` 的 `SimpleApplicationEventMulticaster` Bean 并调用 `setTaskExecutor()`。这种方式影响所有经过该广播器的监听器，粒度比 `@Async` 粗，按需选择。

### 3. 异步异常处理

异步监听器抛出的异常不会传播给发布者。不处理时，异常只会进入执行器的默认处理路径，表现为日志中一条容易被忽略的堆栈。

通过 `AsyncConfigurer` 注册异常处理器：

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            // 记录日志、告警
        };
    };
}
```

异步监听还有第二个限制：方法返回值不会作为后续事件发布。需要发布后续事件时，在方法内注入 `ApplicationEventPublisher` 手动发布。

## 四、事务事件

### 1. 典型用法

事务方法中发布普通事件时，同步监听器在发布调用期间执行，此时事务尚未提交。监听器如果读取数据库，可能读到未提交的数据或读不到数据。

`@TransactionalEventListener` 把监听器绑定到事务阶段：

```java
@TransactionalEventListener
public void onUserRegistered(UserRegisteredEvent event) {
    // 事务提交成功后执行
}
```

默认绑定 `AFTER_COMMIT` 阶段：事务成功提交后触发；事务回滚时不触发。

### 2. 事务阶段与回退执行

`phase` 属性指定触发阶段：

| 阶段               | 触发时机                           |
| ------------------ | ---------------------------------- |
| `AFTER_COMMIT`     | 事务成功提交后（默认）             |
| `AFTER_ROLLBACK`   | 事务回滚后                         |
| `AFTER_COMPLETION` | 事务结束后，无论提交还是回滚       |
| `BEFORE_COMMIT`    | 事务提交前，仍在事务上下文中       |

默认要求发布时存在活动事务，否则监听器不执行。允许无事务时直接执行，设置 `fallbackExecution`：

```java
@TransactionalEventListener(fallbackExecution = true)
public void onUserRegistered(UserRegisteredEvent event) {
}
```

### 3. 组合使用的约束

事务事件监听器可以与 `@Async` 组合，实现提交后异步执行：

```java
@Async
@TransactionalEventListener
public void onUserRegistered(UserRegisteredEvent event) {
    // 事务提交后，在其他线程中执行
}
```

同一个方法上不要同时标注 `@TransactionalEventListener` 和 `@Transactional`。从 Spring Framework 6.1 开始，框架会拒绝这种组合，唯一允许的是 `REQUIRES_NEW` 传播级别。

事务事件监听解决的是"提交后再处理"的时机问题，不解决可靠性问题。`AFTER_COMMIT` 触发后、监听器执行前进程崩溃，事件同样丢失。

## 五、实践注意点

### 1. 事件对象设计

- 使用 `record` 或 `final` 字段，保证不可变。
- 命名使用过去式：`UserRegisteredEvent`、`OrderCancelledEvent`，表达"已经发生的事实"。
- 携带处理所需的全部数据，避免监听器回查数据库。回查会让监听器与发布者的数据状态耦合，事务边界问题也随之而来。

### 2. 异常边界

同步监听器抛出的异常会传播给发布者，中断后续监听器的执行，并可能导致发布方事务回滚。不希望影响主流程的监听器，在方法内部自行捕获异常：

```java
@EventListener
public void sendWelcomeEmail(UserRegisteredEvent event) {
    try {
        mailService.send(event.email());
    } catch (Exception e) {
        log.error("发送欢迎邮件失败: {}", event.userId(), e);
    }
}
```

异步监听器的异常不会传播给发布者，必须配置 `AsyncUncaughtExceptionHandler`，否则失败无人感知。

### 3. 可靠性边界

Spring Event 是进程内机制，不持久化事件，不保证投递。进程崩溃、发布方与监听器之间的任何失败，事件都会丢失。

以下场景超出 Spring Event 的边界，应使用消息队列或事务性发件箱（Transactional Outbox）：

- 跨进程、跨服务的事件投递。
- 要求事件不丢失、可重放。
- 发布方与消费者需要独立的伸缩和故障隔离。

### 4. 不适合使用事件的场景

- 主流程强依赖的处理步骤：支付、扣库存这类失败即整体失败的逻辑，留在直接调用中，让事务和异常语义保持显式。
- 严格依赖执行顺序的链式处理：`@Order` 只约束调用先后，链越长越脆弱，应改为显式的流程编排。
- 高频小粒度事件：监听器匹配和调用有固定开销，高频路径上应评估成本。