# Java注解的本质

## 什么是注解

Java 注解（Annotation）是 JDK 5.0 引入的一种注释机制。它为 Java 代码提供元数据——描述代码本身的信息。注解不直接影响代码的执行；想让注解产生作用，需要有工具去读取它：编译器据此做检查，框架在运行期据此做装配，注解处理器在编译期据此生成新代码。

> 注解和注释的区别：注释写给人看，编译成 class 文件后被丢弃；注解写给编译器、工具和 JVM 看，会保留在 class 文件中，可以被程序读取。
>

注解也常用来替代配置文件。以 Spring 早期的 XML 配置为例，bean 的定义和代码分离，类改名、移动包路径后配置不会同步更新，重构容易出错；注解直接写在代码上，跟着代码走，改名时 IDE 可以一并重构。代价是配置和代码耦合在一起，不改代码就改不了配置。两种做法各有适用场景，注解并没有消灭配置文件。

## 注解的本质

注解的定义语法是 `@interface`：

```java
public @interface MyLog {
}
```

编译之后，`MyLog` 就是一个普通的接口，等价于：

```java
public interface MyLog extends java.lang.annotation.Annotation {
}
```

可以用 `javap` 验证：编译上面的 `MyLog`，执行 `javap MyLog.class`，输出为：

```
public interface MyLog extends java.lang.annotation.Annotation
```

也就是说，注解的本质是一个接口，并且默认继承了 `java.lang.annotation.Annotation` 接口。这个接口里只有一个方法：

```java
Class<? extends Annotation> annotationType();
```

程序中通过反射拿到的注解实例，是 JDK 动态生成的代理对象，背后的调用处理逻辑由 `AnnotationInvocationHandler` 完成，开发者感知不到。

由此可以推出两个限制：注解不能被继承（`extends` 其他注解），也不能用 `new` 实例化，实例只能由系统在读取注解时创建。

## 注解的作用

注解按作用对象分三类。

**编写文档。** Javadoc 生成的 API 文档包含代码中注解的说明，如`@Deprecated` 注解的方法会在文档中明确标出"已弃用"。

**编译检查。** 编译器读取注解并做校验：`@Override` 保证方法确实覆盖了父类方法，写错签名会直接编译报错；`@FunctionalInterface` 保证接口只有一个抽象方法；`@SafeVarargs` 用于抑制可变参数泛型化引起的堆污染警告。这类注解在编译完成后就没有用处了，不会进入 class 文件。

**代码分析。** 注解被工具或程序读取，驱动额外逻辑，这是注解最大的用途。读取方有两个：编译期的注解处理器（如 Lombok 在编译时根据 `@Getter` 生成方法），运行期的反射代码（如 Spring 根据 `@Autowired` 注入依赖，JUnit 根据 `@Test` 找出测试方法）。

## JDK 内置注解

JDK 在 `java.lang` 包下提供了几个常用注解。

`@Override`：标在方法上，要求编译器确认该方法覆盖了父类方法或实现了接口方法。父类删掉了同名方法、或子类方法签名写错时编译报错。这是一个 SOURCE 注解，只服务于编译期。

`@Deprecated`：标在类、方法、字段上，表示该元素已弃用，不建议继续使用。IDE 会对调用处给出删除线提示，编译器在开启相应警告时给出 deprecation 警告（配合 `@SuppressWarnings("deprecation")` 可关闭）。Java 9 起该注解增加了 `since`（从哪个版本弃用）和 `forRemoval`（是否计划移除）两个属性。

`@SuppressWarnings`：告诉编译器忽略指定警告，如 `"unchecked"`（泛型强转警告）、`"deprecation"`（弃用警告）、`"all"`。它只作用于被标注的元素，不影响其他位置的告警。

`@SafeVarargs`（Java 7 引入）：用于声明某个含泛型可变参数的方法不会破坏类型安全，从而免除调用处的堆污染警告。只能用在 static 方法、final 方法、构造方法上（Java 9 起也允许私有方法）。

`@FunctionalInterface`（Java 8 引入）：标在接口上，要求该接口有且只有一个抽象方法（可以有很多默认方法和静态方法），否则编译报错。作用是防止他人修改接口时无意破坏函数式接口的约定。

## 元注解

元注解是加在注解定义上的注解，用来规定注解自身的行为，全部位于 `java.lang.annotation` 包。

`@Target`：规定注解能用在什么地方，取值是 `ElementType` 枚举：`TYPE`（类、接口、枚举）、`FIELD`、`METHOD`、`PARAMETER`、`CONSTRUCTOR`、`ANNOTATION_TYPE`（注解定义本身）、`PACKAGE`、`TYPE_PARAMETER`、`TYPE_USE` 等。不写 `@Target` 表示所有位置可用。

`@Retention`：规定注解保留到哪个阶段，取值是 `RetentionPolicy` 枚举：`SOURCE`（仅源码，编译后丢弃）、`CLASS`（写进 class 文件，运行期 JVM 不保留，默认值）、`RUNTIME`（运行期仍保留，可通过反射读取）。

`@Documented`：让注解出现在 Javadoc 生成的文档里。

`@Inherited`：让注解可以被子类继承——父类标了该注解，子类查询时也能拿到。只对类生效，对接口的实现不生效。

`@Repeatable`（Java 8 引入）：允许同一位置重复标注同一个注解，需要指定一个容器注解来承载多个值。

## 自定义注解

### 定义语法

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyLog {
    String value() default "";
    int level() default 1;
}
```

注解的成员变量以无参数方法的形式声明，名字即成员名，方法名后面的括号不能省略。成员类型只能是基本类型（`int`、`boolean` 等）、`String`、`Class`、枚举、其他注解，以及这些类型的数组，不允许 `Object` 等任意类型。

没有默认值（`default`）的成员，使用注解时必须显式赋值；有默认值的可以省略。默认值不能是 `null`。

### 使用方式

```java
@MyLog(value = "query", level = 2)
public User queryById(long id) {
    ...
}
```

两个便利写法：成员只有一个且名为 `value` 时，可以省略成员名，写成 `@MyLog("query")`；成员是数组且只赋一个值时，可以省略大括号，写成 `tags = "a"`（多个值时必须写 `{"a", "b"}`）。

需要注意，注解本身不含任何逻辑。`@MyLog` 只是给方法打了个标记，记录日志的动作要由读取这个注解的代码完成——这正是下一节的内容。

## 读取注解

注解保留到哪个阶段（`@Retention`），决定了谁能读到它：

- SOURCE：只有编译器看得到，用于编译检查，编译后即丢弃。
- CLASS：写进 class 文件，但 JVM 加载类后不保留，通常供字节码工具使用。
- RUNTIME：运行期仍在，可以通过反射读取。

运行期读取使用反射 API：

```java
Method m = UserService.class.getMethod("queryById", long.class);
if (m.isAnnotationPresent(MyLog.class)) {
    MyLog log = m.getAnnotation(MyLog.class);
    System.out.println(log.value());  // query
}
```

`Class`、`Method`、`Field` 等反射对象都提供 `getAnnotation`、`getAnnotations`、`isAnnotationPresent` 这组方法。

编译期则走另一条路：注解处理器（Annotation Processing Tool，APT）是 javac 的扩展机制，在编译过程中扫描源码里的注解，可以校验代码、生成新的源文件。Lombok、Dagger、MapStruct 等库都基于这套机制工作，编译期生成代码、运行期零开销。

## 注解的实际应用

注解使本来需要大量配置文件和样板代码才能完成的工作，简化为一个或几个标注。典型例子：

- Spring / Spring Boot：`@Component`、`@Autowired`、`@RestController`、`@Transactional`，依赖注入和声明式事务都由框架在运行期反射读取注解后完成，取代了早期大量的 XML 配置。
- JUnit 5：`@Test` 标记测试方法，`@BeforeEach`、`@ParameterizedTest` 等控制测试生命周期。
- Lombok：`@Getter`、`@Data` 等在编译期生成 getter、setter、equals 等方法。

这些框架的共同点是：注解负责声明意图，框架负责读取注解并执行逻辑。理解了"定义—标注—读取"这条链路，阅读任何基于注解的框架源码都会顺畅很多。
