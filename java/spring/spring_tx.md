# Spring事务处理核心原理

## 一、管理体系

Spring 事务管理体系将事务管理行为、事务属性定义和事务执行状态分离：事务管理器执行事务操作，事务定义描述事务运行规则，事务状态记录执行期间的状态与控制信息。

### 1.1 PlatformTransactionManager：事务管理的统一入口

`PlatformTransactionManager` 是 Spring 本地事务管理体系的核心接口，将不同底层资源的事务操作统一为三个动作：

```
public interface PlatformTransactionManager extends TransactionManager {

    TransactionStatus getTransaction(
            TransactionDefinition definition) throws TransactionException;

    void commit(TransactionStatus status) throws TransactionException;

    void rollback(TransactionStatus status) throws TransactionException;
}
```

- `getTransaction()`：按事务定义获取事务。当前无符合条件的事务则创建新事务；已存在事务则按传播行为决定加入、挂起或其他处理方式。
- `commit()`：完成当前事务作用域的提交处理。参与外层事务或已被标记为只能回滚时，不会立即提交底层事务。
- `rollback()`：执行当前事务作用域的回滚处理。参与外层事务时，可能只是将整个事务标记为只能回滚，而不是直接回滚底层数据库事务。

JDBC 用 `Connection` 的 `commit()`、`rollback()`、`setAutoCommit()` 控制本地事务；JPA 通过 `EntityTransaction` 或持久化提供者管理；JTA 通过事务协调接口管理。Spring 不统一底层 API，只统一事务管理契约。

JDBC 场景由 `DataSourceTransactionManager` 管理连接及事务状态，Spring 5.3 起还提供其子类 `JdbcTransactionManager`，差别仅在提交/回滚时把 SQL 异常转换为 `DataAccessException` 体系（与 `JdbcTemplate` 一致）。它们做的不只是转发方法调用，还包括连接获取、事务开启、资源绑定、提交回滚和资源清理。

### 1.2 TransactionDefinition：事务属性的元数据契约

`TransactionDefinition` 不负责开启、提交或回滚事务，只为事务管理器提供处理依据。核心属性有四个：

```
public interface TransactionDefinition {

    int getPropagationBehavior();

    int getIsolationLevel();

    int getTimeout();

    boolean isReadOnly();

    String getName();
}
```

- 传播行为（Propagation）：决定当前方法调用与已有事务的关系——加入、挂起后新建，还是在无事务环境执行。
- 隔离级别（Isolation）：事务对并发数据访问的隔离要求，最终映射到具体资源的能力，通常通过 JDBC 连接设置交给数据库执行。
- 超时时间（Timeout）：事务允许的执行时间上限。超时控制取决于资源实现，Spring 不会在超时点中断正在运行的 Java 代码。
- 只读标记（Read-only）：表达以读取为主的意图。JDBC 下传给连接的只读属性；ORM 下可能据此调整刷新模式。多数数据库不会强制禁止写入。

`@Transactional` 注解、编程式事务模板等入口都通过适配机制形成事务定义，再交给事务管理器。事务属性不会每次调用都完整重作用于底层资源：方法加入已有事务时，隔离级别和超时时间沿用已有物理事务的设置。

### 1.3 TransactionStatus：执行期的生命周期状态维护

`TransactionStatus` 描述一次事务处理过程的执行状态：事务是否完成、当前作用域是否对应新的物理事务、是否已被标记为只能回滚。

```
public interface TransactionStatus
        extends TransactionExecution, SavepointManager, Flushable {

    boolean hasSavepoint();

    void flush();
}
```

继承的 `TransactionExecution` 提供新事务判断、完成状态和回滚标记的读写；`SavepointManager` 提供保存点操作；`Flushable` 用于触发底层会话或连接的刷新。

事务状态不等于数据库事务状态。两个 `PROPAGATION_REQUIRED` 方法各自拥有逻辑事务作用域，却共用同一个物理事务：内层 `TransactionStatus` 表明它不是新事务，仍能参与当前作用域的回滚控制。业务代码调用 `setRollbackOnly()` 后，事务管理器在提交阶段识别该状态并阻止正常提交。

## 二、声明式事务实现

`@Transactional` 标记目标方法，由 Spring AOP 基础设施把元数据与方法调用关联起来，事务拦截器执行事务逻辑。

### 2.1 AOP 基础设施与 TransactionInterceptor

`TransactionInterceptor` 实现 AOP Alliance 的 `MethodInterceptor`（AOP Alliance 是拦截器 API 的公共规范），在目标方法调用前后执行事务逻辑：

```
调用代理对象
    ↓
AOP 选择匹配的拦截器
    ↓
TransactionInterceptor
    ↓
解析事务属性
    ↓
获取 PlatformTransactionManager
    ↓
开启事务或参与已有事务
    ↓
调用目标业务方法
    ↓
根据执行结果提交或回滚
    ↓
返回结果或抛出异常
```

拦截器通过 `TransactionAspectSupport` 的事务执行模板组织流程，核心入口是 `invokeWithinTransaction()`。方法正常返回进入提交处理，抛出异常则按事务属性判断回滚还是继续提交；真正的提交回滚由事务管理器负责。拦截器围绕方法调用组织事务语义，事务管理器围绕底层资源执行事务操作。

### 2.2 代理对象的创建

Spring AOP 用代理对象拦截外部方法调用，只有经过事务拦截链的调用才会执行 `TransactionInterceptor`。

JDK 动态代理基于接口创建代理对象，外部调用经代理处理器进入拦截链再调目标对象。CGLIB 代理通过生成目标类的子类实现，不能代理 `final` 类，也不能通过覆写拦截 `final` 方法。

代理方式由目标类型和配置共同决定：纯 Spring 默认对实现接口的对象用 JDK 代理、无接口的对象用 CGLIB，`proxyTargetClass=true` 强制 CGLIB；Spring Boot 2.0 起该属性默认为 `true`，无论有无接口都用 CGLIB。对事务而言，代理技术不重要，方法调用是否经过事务拦截器才重要。

```
@Service
public class OrderService {

    @Transactional
    public void createOrder() {
        saveOrder();
    }

    @Transactional
    public void saveOrder() {
        // 写入订单数据
    }
}
```

外部通过容器取得的代理对象调用 `createOrder()` 时，事务拦截器介入。`createOrder()` 内部的 `this.saveOrder()` 是目标对象上的普通方法调用，不经过代理，`saveOrder()` 的注解不会独立触发一次事务拦截。若 `createOrder()` 已开启事务，内部调用仍运行在当前事务中，只是注解不会被重新解析执行。

### 2.3 基础设施切面与目标定位

事务注解解析与代理创建发生在 Bean 初始化和 AOP 基础设施配置阶段，不在每次调用时扫描注解。`@EnableTransactionManagement` 通过 `@Import` 引入两个配置：

- `AutoProxyRegistrar`：注册自动代理创建器 `InfrastructureAdvisorAutoProxyCreator`，识别需要增强的 Bean 并创建代理。
- `ProxyTransactionManagementConfiguration`：装配 `AnnotationTransactionAttributeSource`（从方法和类解析事务属性）、`TransactionInterceptor` 和 `BeanFactoryTransactionAttributeSourceAdvisor`。

Advisor 是切点与增强逻辑的组合：切点判断哪些类和方法需要增强，拦截器执行事务处理。代理创建器发现目标 Bean 存在匹配的事务增强时，把事务拦截器纳入其代理调用链，此后外部调用代理对象才触发事务。

属性解析规则：方法上的注解优先于类上的注解；接口方法与实现类方法都标注时，按属性源的解析顺序取先命中的一份。

### 2.4 事务不生效的典型场景

| 场景 | 原因 | 对策 |
| --- | --- | --- |
| 类内方法互调 | 内部调用不经过代理 | 注入自身代理或拆成独立 Bean |
| `final` 方法或 `final` 类 | CGLIB 无法覆写或继承 | 去掉 `final`，或改用 JDK 代理 |
| 非 `public` 方法 | 代理模式只对 `public` 方法拦截 | 注解移到 `public` 入口方法 |
| 异常被捕获后正常返回 | 拦截器看到的是正常返回 | 继续向外抛，或显式 `setRollbackOnly()` |
| 方法内新开线程做数据库操作 | 事务资源绑定在线程上，新线程取不到 | 新线程的操作不纳入本事务 |
| 对象未经 Spring 容器管理 | 根本没有代理 | 交给容器管理 |

## 三、事务语义：传播、隔离与回滚规则

### 3.1 传播机制：线程挂起、恢复与嵌套事务

传播行为描述的是事务作用域与已有事务的关系，不是简单地描述是否执行 `BEGIN` 或 `COMMIT`。

`PROPAGATION_REQUIRED`：当前线程已有事务则加入，没有则新建。外层与内层各有逻辑事务作用域，共用一个物理事务，内层正常返回不会独立提交数据库事务。

```
外层方法：开启事务 T1
    ↓
内层方法：REQUIRED
    ↓
参与事务 T1
    ↓
内层方法结束
    ↓
外层方法继续执行
    ↓
提交或回滚 T1
```

`PROPAGATION_REQUIRES_NEW`：挂起外层事务，开启独立的物理事务，内层结束后恢复外层。

```
外层事务 T1
    ↓
挂起 T1
    ↓
开启内层事务 T2
    ↓
提交或回滚 T2
    ↓
恢复 T1
    ↓
继续执行外层逻辑
```

`REQUIRES_NEW` 需要额外的数据库连接：外层连接上挂着未结束的事务，内层独立事务需要另一条能独立提交回滚的连接。

`PROPAGATION_NESTED`：在现有事务内建立嵌套作用域，JDBC 下通过 `Savepoint` 实现。内层回滚只回到保存点，不波及外层；内层正常结束不等于数据库事务已提交，最终提交仍由外层物理事务控制。当前没有事务时，`NESTED` 会新建事务，而不是凭空创建保存点。

其余传播行为：

- `SUPPORTS`：有事务则参与，没有则以非事务方式执行。
- `MANDATORY`：要求已有事务，没有则抛异常。
- `NOT_SUPPORTED`：以非事务方式执行，已有事务则先挂起。
- `NEVER`：要求没有事务，已有事务则抛异常。

传播行为是事务管理器对作用域的组织规则。不同事务管理器在挂起、恢复、嵌套上的能力不同，不能当成所有底层资源都能无条件实现的操作。

### 3.2 隔离级别与底层数据库连接

Spring 的 `Isolation` 枚举：

| 级别 | 含义 |
| --- | --- |
| `DEFAULT` | 采用底层数据库的默认隔离级别 |
| `READ_UNCOMMITTED` | 读未提交 |
| `READ_COMMITTED` | 读已提交 |
| `REPEATABLE_READ` | 可重复读 |
| `SERIALIZABLE` | 串行化 |

JDBC 本地事务通过 `Connection.setTransactionIsolation()` 把隔离级别映射到底层连接，由数据库执行并发控制。实际效果取决于数据库实现：不同数据库对快照读取、锁定读取和范围锁的处理不同，同名隔离级别的并发行为未必一致。

加入已有事务时，隔离级别不会单独生效。外层以 `READ_COMMITTED` 开启事务，内层声明 `REPEATABLE_READ` 且使用 `REQUIRED` 时，内层加入的是外层物理事务，Spring 不会为其重开数据库事务或切换连接隔离级别。`AbstractPlatformTransactionManager` 开启 `validateExistingTransaction` 后，可对这类不兼容配置做校验并报错，而不是默默忽略。

隔离级别是事务定义的属性，但参与已有事务的方法不能独立决定实际使用的隔离级别。

### 3.3 回滚规则：异常体系与 rollbackFor 的匹配逻辑

默认只对 `RuntimeException` 和 `Error` 回滚，受检异常不回滚。这是 Spring 的规则，不是数据库对 Java 异常类型的要求。

```
@Transactional
public void createOrder() throws Exception {
    saveOrder();
    throw new Exception("创建失败");
}
```

这段代码按默认规则不会回滚。让它回滚需要显式声明 `@Transactional(rollbackFor = Exception.class)`。

解析后，回滚规则封装进事务属性对象；拦截器捕获异常后按属性判断，`RuleBasedTransactionAttribute` 负责匹配，规则来自 `rollbackFor`、`noRollbackFor` 及其字符串形式。多条规则同时匹配时取最具体的一条（按异常继承深度判断）；没有自定义规则命中时用默认判断。

拦截器只对它实际观察到的异常和事务状态做判断。方法内捕获异常并正常返回时，拦截器看到的是正常返回，不会回滚，除非事务已被显式标记为只能回滚：

```
@Transactional
public void createOrder() {
    try {
        saveOrder();
        throw new RuntimeException("业务失败");
    } catch (RuntimeException ex) {
        // 异常被吞掉
    }
}
```

## 四、执行机制：资源同步与生命周期闭环

### 4.1 TransactionSynchronizationManager 与 ThreadLocal 资源绑定

`TransactionSynchronizationManager` 维护当前线程关联的事务资源、同步回调和事务上下文，不直接执行事务操作。事务管理器开启事务时获取连接，包装为 `ConnectionHolder` 绑定到当前线程：

```
当前线程
    │
    └── TransactionSynchronizationManager
            │
            ├── 资源映射
            │     └── DataSource → ConnectionHolder
            │
            ├── 事务同步回调
            │
            └── 事务名称、只读状态、
                  隔离级别、实际事务状态等信息
```

资源映射以资源工厂对象为键，JDBC 下即 `DataSource → ConnectionHolder`。同一线程的 DAO 通过 Spring 的资源获取机制访问同一 `DataSource` 时，复用已绑定的连接，不必层层传参即可参与同一个物理事务。线程切换后，新线程不会取得原线程绑定的资源；两个线程即使用相同的事务管理器和数据源，也不共享同一个本地事务。

### 4.2 DataSourceUtils：连接的安全获取与生命周期托管

`DataSourceUtils` 识别当前线程是否已绑定指定 `DataSource` 对应的事务连接：

```
Connection connection =
        DataSourceUtils.getConnection(dataSource);
```

已绑定则直接返回事务连接；未绑定则从数据源获取并处理使用与释放。直接调用 `dataSource.getConnection()` 没有这种识别能力。多个 DAO 都走这套机制（`JdbcTemplate` 即如此），就共享同一条事务连接，处于同一个物理事务中。

释放同样区分归属：仍由事务管理器管理的连接，DAO 操作结束时不会真正关闭，由事务管理器在事务结束时统一清理。绕过这套机制自行获取的连接，复用不到当前事务连接。

### 4.3 事务生命周期闭环：doBegin、doCommit、doRollback

`AbstractPlatformTransactionManager` 实现通用逻辑：传播行为处理、事务状态创建、提交回滚协调、同步回调和资源清理；具体事务管理器负责底层资源操作。JDBC 本地事务的典型过程：

```
TransactionInterceptor
    ↓
AbstractPlatformTransactionManager
    ↓
getTransaction()
    ↓
判断已有事务与传播行为
    ↓
创建事务状态
    ↓
doBegin()
    ↓
获取连接、配置事务属性、绑定资源
    ↓
执行目标业务方法
    ↓
commit() 或 rollback()
    ↓
doCommit() 或 doRollback()
    ↓
执行事务完成回调
    ↓
清理资源并恢复线程上下文
```

`doBegin()` 是开启底层事务的扩展点：获取连接、设置自动提交、应用隔离级别和只读属性、把资源绑定到线程。方法正常结束时，拦截器调用 `commit()`；后者先检查事务状态、回滚标记、是否新事务、是否参与外层事务，真正负责提交的作用域才调用 `doCommit()`（JDBC 下即连接的 `commit()`）。参与外层事务的作用域回滚时只设置全局回滚标记，由外层结束时统一处理。物理事务结束后，执行完成回调、解绑线程资源、恢复连接原状并释放连接。