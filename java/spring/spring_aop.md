# Spring的核心AOP

AOP（Aspect Oriented Programming），即面向切面编程，是一种编程思想。主要用来完善OOP在处理通用非核心功能方面的不足之处，让OOP只关注对象的核心业务功能。常见的用在日志、事务、异常、安全等各方面。

AOP 中的基本单元是Aspect(切面)，就是AOP利用"横切"的技术，切入到对象内部，将多个类的公共行为封装到一个可重用模块中。

AOP将多个应用对象分成两个部分：核心关注点和横切关注点。业务处理的主要流程是核心关注点，与之关系不大的部分是横切关注点。

## AOP的术语

- 通知（Advice）: 实际业务对象方法之前或之后执行的方法（增强）。通知描述了切面何时执行以及如何执行公共代码。
- 连接点（join point）: 连接点表示应用执行过程中能够插入切面的一个点，这个点可以是方法的调用、异常的抛出。在 Spring AOP 中，连接点总是被拦截到的方法。
- 切点（PointCut）: 可以插入执行公共代码的一组连接点。
- 切面（Aspect）: 切面是通知和切点的结合。
- 引入（Introduction）：引入允许我们向现有的类添加新的方法或者属性。
- 织入（Weaving）: 将公共代码添加到目标对象中，并创建一个被增强的对象，这个过程就是织入。

通知分为前置、后置、异常、最终、环绕通知五类：

| 通知           | 描述                                                         |
| :------------- | :----------------------------------------------------------- |
| Before         | 前置通知，在一个方法执行之前，执行通知。                     |
| After          | 后置通知，在一个方法执行之后，不考虑其结果，执行通知。       |
| AfterReturning | 返回后通知，在一个方法执行之后，只有在方法成功完成时，才能执行通知。 |
| AfterThrowing  | 抛出异常后通知，在一个方法执行之后，只有在方法退出抛出异常时，才能执行通知。 |
| Around         | 环绕通知，在建议方法调用之前和之后，执行通知。               |

## Spring AOP的实现

AOP是在不修改源代码的前提下实现对业务组件加入新的通用功能，那么其本质是由框架去修改业务组件的指定方法的源代码。有两类做法来实现AOP，一个是静态实现，在编译阶段进行修改源代码；另外一个是动态实现，在运行阶段动态生成代理对象。

Spring的AOP框架利用是动态代理技术实现，分为两种规则，由框架自动选择：

- 接口的代理，默认使用JDK动态代理(Dynamic Proxy)来创建。
- 非接口的代理，会自动使用CGLIB创建子类，也可以强制指定。

Spring AOP通过Spring IoC提供一个简单的AOP实现，只能用于Spring容器管理的beans，并且使用了AspectJ框架的语法，也支持@AspectJ 的 AOP 风格，所以完全兼容。

但是Spring AOP并没有实现AspectJ的完整功能，需要使用完整的AOP的时候，还需要使用AspectJ框架，在spring-aspects做了集成。相比Spring AOP，AspectJ是静态实现，需要编译器在编译阶段做切面，性能要提高很多。

利用Spring AOP框架，做切面编程时，则只需要做以下事情，其他的代理由框架自动生成：

- 确定需要切面的业务组件。
- 定义切入点，一个切入点可能横切多个业务组件。
- 编写通用的功能代码，也即增强处理。

## Spring AOP组件

IOC容器启动时，会扫描xml配置文件，根据spring.handlers中配置的aop命名空间处理类，在解析BeanDefination的DefaultBeanDefinitionDocumentReader会使用此类配置的信息。

- AopNamespaceHandler，解析xml配置文件中的aop命名空间下的标签，各元信息解析器的注册入口。
- BeanDefinitionParser，Bean元信息解析，由AopNamespaceHandler配置。其中aop:config标签（XML风格）用ConfigBeanDefinitionParser来解析；aop:aspectj-autoproxy标签（@AspectJ风格）则使用 AspectJAutoProxyBeanDefinitionParser 解析。它们又分别注册了AspectJAwareAdvisorAutoProxyCreator和AnnotationAwareAspectJAutoProxyCreator两种创建代理的方式。

SpringBoot根据@Configuration会找到自动配置类。

- AopAutoConfiguration，SpringBoot自动配置类，主要解析注解@EnableAspectJAutoProxy。从而加载AspectJAutoProxyRegistrar类。完成对AnnotationAwareAspectJAutoProxyCreator的Bean注册，也确定了是否使用cglib。

Spring也提供了编程API，以编码方式配置AOP。

- ProxyFactory，提供的AOP API，让开发者以编程的方式配置Spring AOP。

Spring AOP的生成代理类的组件核心是AbstractAutoProxyCreator，并有不同的实现子类，分别处理风格的代理生成逻辑。

- AbstractAutoProxyCreator，是AOP的一个核心类，实现了代理创建的逻辑，使用AOP代理包装每个合格的bean，并在调用bean本身之前委派给指定的拦截器。

- AspectJAwareAdvisorAutoProxyCreator，自动代理创建者，为特殊的bean构建AOP代理。是AbstractAutoProxyCreator的实现子类。

- AnnotationAwareAspectJAutoProxyCreator，是用于处理当前应用程序上下文中的所有AspectJ注释方面以及Spring Advisor。是AspectJAwareAdvisorAutoProxyCreator的子类。

Spring AOP的两种主要方式来生成代理对象，对应的两个实现类。

- CglibAopProxy，使用Cglib来生成具体代理子类。

- JdkDynamicAopProxy，使用JDK动态代理生成具体的代理类，是InvocationHandler的实现类。

注：InvocationHandler是JDK动态代理的核心，生成的代理对象的方法调用都会委托到InvocationHandler.invoke()方法。

