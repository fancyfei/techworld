# Spring AOP 的具体应用

使用 Spring AOP 只有三步：声明切面、定义切点、编写通知，其余交给代理。

## 注解式 API

### 声明切面

```java
@Aspect                          // 声明为切面
@Component                        // 交给 IOC 容器管理
public class LogAspect { }
```

XML 等价写法（了解即可）：

```xml
<aop:config>
    <aop:aspect ref="logAspect">
        <aop:before method="before" pointcut="execution(* com.example.service..*(..))"/>
    </aop:aspect>
</aop:config>
```

### 切点表达式

```java
// 匹配 service 包及子包下所有类的所有方法
@Pointcut("execution(* com.example.service..*.*(..))")
public void serviceLayer() { }

// 匹配标注了 @OperLog 的方法
@Pointcut("@annotation(com.example.annotation.OperLog)")
public void operLog() { }

// 匹配类上标注了 @Service 的所有方法
@Pointcut("@within(org.springframework.stereotype.Service)")
public void serviceBean() { }

// 匹配第一个参数为 String 的方法
@Pointcut("args(String, ..)")
public void stringFirstArg() { }

// 按 Bean 名称匹配
@Pointcut("bean(orderService*)")
public void orderBeans() { }

// 组合：service 包中且标注 @OperLog 的方法
@Pointcut("serviceLayer() && operLog()")
public void serviceOperLog() { }
```

execution 语法各段含义：

```java
execution(修饰符 返回值 包名.类名.方法名(参数) throws 异常)
// 例：execution(public * com.example.service.*.*(..))
//              ↑public 可省略     ↑一层包  ↑类 ↑方法 ↑任意参数
```

### 五种通知

**Before**通知，方法执行前执行，`JoinPoint` 可拿目标信息：

```java
@Before("serviceLayer()")
public void before(JoinPoint jp) {
    String method = jp.getSignature().getName();   // 方法名
    Object[] args = jp.getArgs();                  // 入参
}
```

**AfterReturning**通知，方法正常返回后执行，`returning` 拿返回值（注意参数名要一致）：

```java
@AfterReturning(pointcut = "serviceLayer()", returning = "result")
public void afterReturning(Object result) { }
```

**AfterThrowing**通知，方法抛异常时执行，`throwing` 拿异常：

```java
@AfterThrowing(pointcut = "serviceLayer()", throwing = "ex")
public void afterThrowing(Exception ex) { }
```

**After**通知，无论正常返回还是抛异常都执行，相当于 finally：

```java
@After("serviceLayer()")
public void after() { }
```

**Around**通知，包裹整个方法调用，能力最强：

```java
@Around("serviceLayer()")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    Object[] args = pjp.getArgs();
    args[0] = args[0].toString().trim();      // 修改入参
    Object result = pjp.proceed(args);        // 放行，不传参则按原参数执行
    // pjp.proceed() 不调用，目标方法不会执行
    return result;                            // 可替换为目标方法之外的返回值
}
```

五种通知在方法正常返回时的执行顺序：

```java
@Around 前半 → @Before → 目标方法 → @AfterReturning → @After → @Around 后半
// 方法抛异常时：@AfterReturning 不执行，换成 @AfterThrowing，@After 照常执行
```

### 通知获取目标信息

| API | 说明 |
| :--- | :--- |
| `JoinPoint.getSignature()` | 方法签名（名、所在类等） |
| `JoinPoint.getArgs()` | 入参数组 |
| `JoinPoint.getTarget()` | 目标对象（被代理的原始 Bean） |
| `ProceedingJoinPoint.proceed()` | 放行目标方法，可传入修改后的参数 |
| `Annotation` 直接入参 | 匹配 `@annotation(x)` 时注入注解本身 |

### 多切面执行顺序

```java
@Order(1)   // 数值越小越先执行，环绕通知表现为洋葱模型：先进后出
@Aspect
@Component
public class FirstAspect { }
```

## 编程式 API

不依赖注解和配置，代码里直接生成代理：

```java
ProxyFactory factory = new ProxyFactory(new OrderService());
factory.addAdvice(new MethodInterceptor() {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // 环绕逻辑
        return invocation.proceed();
    }
});
OrderService proxy = (OrderService) factory.getProxy();
```

已有 @Aspect 切面的类，用 AspectJProxyFactory 直接套：

```java
AspectJProxyFactory factory = new AspectJProxyFactory(target);
factory.addAspect(LogAspect.class);
OrderService proxy = factory.getProxy();
```

## 典型应用方式

### 日志与耗时统计

```java
@Around("execution(* com.example.controller..*(..))")
public Object log(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();
    log.info("{} 入参: {}", pjp.getSignature(), pjp.getArgs());
    try {
        Object result = pjp.proceed();
        log.info("{} 出参: {}, 耗时: {}ms",
                pjp.getSignature(), result, System.currentTimeMillis() - start);
        return result;
    } catch (Throwable e) {
        log.error("{} 异常", pjp.getSignature(), e);
        throw e;
    }
}
```

### 自定义注解 + 切面

注解定义权限要求，切面在方法前校验：

```java
@Target(ElementType.METHOD)
public @interface RequirePerm { String value(); }

@Before("@annotation(requirePerm)")
public void check(RequirePerm requirePerm) {
    if (!currentUser().hasPerm(requirePerm.value())) {
        throw new AccessDeniedException("无权限");
    }
}
```

使用处一行声明：

```java
@RequirePerm("order:delete")
public void deleteOrder(Long id) { }
```

限流、防重复提交是同一套路：注解携带元数据（QPS、过期秒数），切面在 `proceed()` 前做检查，不通过则抛异常。

### 返回值处理

脱敏——方法返回后改写字段：

```java
@AfterReturning(pointcut = "@annotation(sensitive)", returning = "result")
public void mask(Object result) {
    // 反射遍历 result 中带 @Masked 的字段，按类型打码
    // 手机号: 138****5678
}
```

异常转换——DAO 层异常转为业务异常：

```java
@AfterThrowing(pointcut = "execution(* com.example.dao..*(..))", throwing = "ex")
public void translate(DuplicateKeyException ex) {
    throw new BizException("记录已存在", ex);
}
```

## 框架内置的 AOP 应用

日常最常见的几个注解本身就是 AOP 代理实现的，理解了切面机制就能看懂它们的限制：

| 注解 | 织入的逻辑 | 代理方式 |
| :--- | :--- | :--- |
| @Transactional | 方法进入时开启事务，退出时提交或回滚 | AOP 代理 |
| @Cacheable | 方法执行前查缓存，命中直接返回 | AOP 代理 |
| @Async | 方法调用切到线程池异步执行 | AOP 代理 |
| @Scheduled | 定时触发目标方法 | 定时调度器回调 |

以 @Transactional 为例，代理的环绕逻辑等价于：

```java
// 伪代码：事务切面的环绕通知
tx = tm.begin();            // 根据 @Transactional 属性开启事务
try {
    result = targetMethod();   // 调用真实业务方法
    tx.commit();
} catch (RuntimeException e) {
    tx.rollback();             // 默认只对运行时异常回滚
    throw e;
}
```

## 使用时的注意点

- 切面方法内部不要用 `this.xxx()` 或同类内部调用，否则不经过代理，所有增强失效（如 @Transactional 加在内部调用方法上不生效）。
- final 类、final 方法、private 方法无法被 CGLIB 代理，切面对其静默不生效。
- 一个方法匹配多个切面时，用 `@Order` 控制切面执行顺序，数值越小越先执行。
- 代理方式不确定时打印 `bean.getClass().getName()`：JDK 代理显示 `$Proxy` 开头，CGLIB 显示 `$$EnhancerBySpringCGLIB`。