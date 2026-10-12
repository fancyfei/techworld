# Spring Bean 的具体应用

Bean 的管理机制最终体现在注册、注入、作用域、生命周期、扩展五个环节。

## 声明与注册 Bean

注解扫描是目前的普遍做法，配合构造型注解声明组件：

```java
@Service
public class OrderService {
    // 业务逻辑
}

@Configuration
@ComponentScan("com.example.order")
public class AppConfig {
}
```

第三方库中的类无法用注解标注，用 @Bean 方法注册：

```java
@Configuration
public class DataConfig {

    @Bean
    public DataSource dataSource() {
        return new HikariDataSource(); // 精简伪代码：实际读取连接配置
    }
}
```

遗留 XML 配置等价写法：

```xml
<bean id="orderService" class="com.example.order.OrderService"/>
```

按环境注册不同实现：

```java
@Bean
@Profile("test")
public DataSource h2DataSource() { /* ... */ }

@Bean
@Profile("prod")
public DataSource mysqlDataSource() { /* ... */ }
```

在 Spring Boot 中还可以用 `@ConditionalOnProperty("feature.x.enabled")` 按配置开关注册。

## 注入依赖

构造器注入是目前的普遍做法，依赖可声明为 final，对象创建后即处于完整状态。只有一个构造器时，Spring 4.3 起可省略 @Autowired：

```java
@Service
public class OrderService {

    private final OrderRepository repository;
    private final MailClient mailClient;

    public OrderService(OrderRepository repository, MailClient mailClient) {
        this.repository = repository;
        this.mailClient = mailClient;
    }
}
```

字段与 setter 注入写法更简，适合可选依赖和测试替换：

```java
@Autowired
private MailClient mailClient;

@Autowired
public void setMetrics(MetricsClient metrics) { // 可选依赖
    this.metrics = metrics;
}
```

### 可选依赖

依赖不存在时容器默认抛异常，三种声明可选的方式：

```java
@Autowired(required = false)
private MetricsClient metrics; // 多个候选存在时仍会抛 NoUniqueBeanDefinitionException

public OrderService(@Nullable MetricsClient metrics) { /* ... */ }
// @Nullable 为 org.springframework.lang.Nullable，Spring 5.0 起支持注入点标注

@Autowired
private Optional<MailClient> mailClient; // Java 8+ 等价写法
```

需要延迟到使用时才解析，或想附带默认实现，用 ObjectProvider（Spring 4.3 引入，回调式 API 为 5.0 新增）：

```java
@Component
public class ReportJob {

    private final ObjectProvider<HeavyService> service;

    ReportJob(ObjectProvider<HeavyService> service) {
        this.service = service;
    }

    void run() {
        service.ifAvailable(HeavyService::warmUp);       // 存在才回调
        HeavyService s = service.getIfAvailable(HeavyService::createDefault); // 缺省兜底
    }
}
```

### 多个候选与集合注入

容器中存在多个同类型候选时，用 @Qualifier 指定，或给其中一个实现标 @Primary：

```java
public OrderService(@Qualifier("smsNotifier") Notifier notifier) { /* ... */ }
```

一个接口有多个实现、需要全部注入时，直接声明集合：

```java
public NotificationCenter(List<Notifier> notifiers) { /* ... */ }
// List 按 @Order 注解排序

public NotificationCenter(Map<String, Notifier> notifiers) { /* ... */ }
// Map 的 key 是 Bean 名称（{"smsNotifier": ..., "emailNotifier": ...}），适合做策略路由
```

### 延迟注入与方法注入

@Lazy 注入的是代理，首次调用才真正创建对象，也用于打破循环依赖：

```java
public ReportController(@Lazy ReportBuilder builder) { /* ... */ }
```

@Lookup 方法注入：抽象方法由容器生成实现，每次调用返回新的 prototype 实例，是 prototype 注入单例陷阱的另一种解法（见"作用域"一节）。通过 CGLIB 生成子类代理，类与方法不能声明为 final：

```java
@Component
public abstract class OrderProcessor {

    @Lookup
    protected abstract ReportBuilder newBuilder(); // 每次调用返回新实例
}
```

### 配置值注入

@Value 注入配置项与 SpEL 表达式：

```java
@Value("${order.timeout:30}")
private int timeoutSeconds; // 带默认值

@Value("#{systemProperties['user.region']}")
private String region;
```

遗留 XML 配置中的注入写法：

```xml
<bean id="orderService" class="com.example.OrderService">
    <constructor-arg ref="orderRepository"/>
    <property name="mailClient" ref="mailClient"/>
</bean>
```

三个注入注解的匹配规则不同，混用时需要留意：

| 注解 | 来源 | 匹配规则 |
| --- | --- | --- |
| @Autowired | Spring | 默认按类型；可配 required = false 表示可选 |
| @Resource | JSR-250 | 默认按名称，找不到再按类型 |
| @Inject | JSR-330 | 按类型，需额外引入 javax.inject 依赖 |

构造器循环依赖在启动时直接抛出 BeanCurrentlyInCreationException，容器无法自动解决，两种处理手段：

```java
// 手段一：对一方加 @Lazy，先注入代理，首次调用时才创建真实对象
public OrderService(@Lazy InventoryService inventoryService) { /* ... */ }

// 手段二：把其中一环改为 setter/字段注入，让另一环先完成实例化
```

## 按场景选择作用域

singleton 是默认作用域，无状态服务直接复用单例即可；有状态、非线程安全的对象用 prototype；Web 环境的 ApplicationContext 额外提供 request、session、application、websocket 四种作用域。

prototype 注入 singleton 是一个常见陷阱：单例只在注入那一刻取一次 prototype，此后永远持有同一个实例。两种解法，任选其一：

```java
// 解法一：代理模式，每次调用代理方法时向容器取新实例
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
@Component
public class ReportBuilder { /* ... */ }
// 解法二：注入 ObjectProvider，每次从容器中现取（写法见"注入依赖"一节）
```

Web 作用域 Bean 注入单例时同样需要代理：

```java
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
@Component
public class Cart { /* 保存当前用户的购物车 */ }
```

## 生命周期回调

初始化与销毁逻辑优先用 JSR-250 注解，与 Spring 接口解耦：

```java
@PostConstruct
void init() { // 依赖注入完成后执行：建立连接、预热缓存
}

@PreDestroy
void cleanup() { // 容器关闭前执行：释放连接、flush 缓冲区
}
```

Spring 5 及更早版本引自 `javax.annotation`，Spring 6 起改为 `jakarta.annotation`，升级 Boot 3 时需要同步修改 import。

等价的接口与配置方式仍广泛存在于既有代码中：

```java
public class Client implements InitializingBean, DisposableBean {
    public void afterPropertiesSet() { /* ... */ }
    public void destroy() { /* ... */ }
}
<bean id="client" class="com.example.Client" init-method="init" destroy-method="close"/>
```

同一 Bean 上多种方式并存时的执行顺序：@PostConstruct → InitializingBean.afterPropertiesSet() → init-method；销毁阶段对称反向执行。

应用启动后执行一次的逻辑（数据初始化、注册监听）用 Runner 接口，比 @PostConstruct 更靠后，依赖的 Bean 已全部就绪：

```java
@Component
public class SeedDataRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        // 精简伪代码：插入初始数据
    }
}
```

## 扩展与高级用法

非 Spring 管理的对象（如静态工具方法、遗留单例）需要拿到容器中的 Bean 时，常见做法是通过 Aware 接口保存上下文：

```java
@Component
public class SpringContextHolder implements ApplicationContextAware {

    private static ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        context = ctx;
    }

    public static <T> T getBean(Class<T> type) {
        return context.getBean(type);
    }
}
```

BeanPostProcessor 适合做横切处理，为特定 Bean 统一包装逻辑：

```java
// 场景：给所有标注 @LogCost 的 Bean 加方法耗时日志
public class LogCostBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean.getClass().isAnnotationPresent(LogCost.class)) {
            // 精简伪代码：创建代理，方法调用前后记录耗时并打印
            return wrapWithTimingProxy(bean);
        }
        return bean;
    }
}
```

构建过程复杂的对象（需要多步装配、依赖外部资源）用 FactoryBean，MyBatis 的 SqlSessionFactoryBean、Quartz 的 SchedulerFactoryBean 都是这种用法：

```java
public class ApiClientFactoryBean implements FactoryBean<ApiClient> {

    @Override
    public ApiClient getObject() {
        // 精简伪代码：读配置、建连接池、装配拦截器，返回客户端
    }

    @Override
    public Class<?> getObjectType() {
        return ApiClient.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

注入时拿到的直接是 getObject() 返回的 ApiClient，而不是工厂本身；需要工厂时通过 `&beanName` 名称获取。