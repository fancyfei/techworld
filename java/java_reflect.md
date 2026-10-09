# Java 反射基本原理

## 什么是反射？

**反射就是让 Java 程序在运行时动态获取类的信息，或者动态调用对象的方法和属性。**

静态语言通常在编译期就把类、方法、属性都固定好了。而反射机制给 Java 开了个后门，允许程序在跑起来之后再去动态加载某个类、看这个类里有什么、强行修改它的字段或者调用它的方法。

## Class 对象从哪来

反射的一切都围绕 `Class` 对象展开，而 `Class` 对象是在类加载阶段由 JVM 自动创建的：类加载器把 `.class` 字节码读进内存后，JVM 会在方法区（Java 8 起在元空间）里为这份字节码生成一个唯一的 `java.lang.Class` 实例。同一个类在同一个类加载器下，全 JVM 只有一个 `Class` 对象，这就是"类本身也是对象"这句话的来源。

类加载大致分三个阶段：加载（读字节码、生成 Class 对象）、链接（验证、准备、解析）、初始化（执行 `<clinit>`，也就是静态变量赋值和静态代码块）。反射 API 的行为和这三个阶段直接相关——后面会看到，`Class.forName()` 会走到初始化，而 `User.class` 只拿到对象、不触发初始化。

## 核心 API 与基础用法

### 1. 获取 Class 对象

拿到 `Class` 对象是反射的第一步，通常有三种方式：

- `Class.forName("com.example.User")`：通过全类名加载，最常用（会触发类的静态初始化）。
- `User.class`：直接拿类名点 class，安全高效（不会触发类初始化）。
- `user.getClass()`：有了实例对象之后再拿。

三种方式的区别就来自上一节的类加载机制：`forName` 会走完加载、链接、初始化三个阶段，所以静态代码块会执行；`User.class` 只是编译期就把 Class 对象引用写死进字节码，类可能还没被加载，自然也不会初始化。

### 2. 构造方法、属性与方法操作

- **实例化对象**：`Constructor.newInstance(...)`（Java 9 之后废弃了 `Class.newInstance()`，统一改用构造器方式）。
- **读写属性**：`Field.get(instance)` / `Field.set(instance, value)`。
- **调用方法**：`Method.invoke(instance, args...)`，调用静态方法时第一个参数传 `null`。

有一组方法名非常容易踩坑：`getFields()` / `getMethods()` 只返回 **public** 成员（且包含从父类继承来的），而 `getDeclaredFields()` / `getDeclaredMethods()` 返回本类 **声明的全部** 成员（含 private，但不含继承的）。想拿私有成员，必须用带 `Declared` 的版本。

### 3. 强制访问 private

遇到私有方法或私有变量，直接调 `setAccessible(true)` 强行破局。不过在 Java 9 引入模块化（JPMS）之后，跨模块的暴力反射会被限制——如果目标类所在的模块没有向你开放包，会抛 `InaccessibleObjectException`，`setAccessible(true)` 也救不回来，只能在模块声明里加 `opens` 或者加 JVM 参数 `--add-opens` 解决。

## 底层原理

反射能跑，底层核心在于 `Method.invoke()`。这块的实现随 JDK 版本变过一次，分界线是 **JDK 18**。

**JDK 17 及之前：委派 + 膨胀（Inflation）**

1. **委派模式**：`Method.invoke()` 内部把活交给了 `MethodAccessor` 接口去干。
2. **膨胀机制**：
   - **前 15 次调用**：默认使用 **JNI（Native 调用）**。这样做启动快，不需要生成额外的类，但每次调用都要走 Native 层，执行效率低。
   - **第 16 次及以后**：JVM 觉得这个方法是热点，就会在内存里动态生成一个全新的字节码类（`GeneratedMethodAccessorXXX`），直接强转类型并调用原方法。这就是"膨胀"。之后再调用，就变成了纯粹的 Java 方法调用，性能大大提升。

**JDK 18 及以后：改用 MethodHandle 实现**

JDK 18 交付的 JEP 416 用 MethodHandle 重写了核心反射：`Method.invoke()`、`Constructor.newInstance()`、`Field.get/set` 底层不再生成膨胀类，而是直接委派给方法句柄。旧的膨胀实现暂时保留，加 `-Djdk.reflect.useDirectMethodHandle=false` 可以切回去，但官方已明确它会在未来版本移除。

所以网上讲的"前 15 次慢、16 次起飞"只适用于 JDK 17 及之前的 HotSpot；"膨胀""GeneratedMethodAccessor"这些概念作为历史实现值得了解，但写在 JDK 18+ 的项目里不会复现。

### 反射的副作用与坑

1. **性能慢**：每次调用都有装箱拆箱和安全检查的开销，成员查找（`getMethod` 之类的字符串匹配）也不便宜；JIT 虽然能优化最终的调用路径，但整体仍明显慢于直接调用。
2. **破坏封装**：调用 `setAccessible(true)` 可以无视 `private`，分分钟破坏单例模式或修改不可变对象。
3. **绕过泛型**：泛型只存在于编译期，运行时利用反射能直接往 `List<String>` 里放 `Integer`。
4. **抛弃编译检查**：类名或方法名写错，编译时完全不报错，一跑起来直接抛 `ClassNotFoundException` 或 `NoSuchMethodException`。

## 实战场景与替代

### 1. 核心应用场景

- **IOC / DI**：Spring 根据配置文件或注解，用反射实例化 Bean 并注入依赖。
- **ORM 框架**：MyBatis / Hibernate 查出数据后，用反射把 `ResultSet` 塞进 POJO 的私有字段里。
- **序列化**：Jackson / Fastjson 动态解析对象属性，拼装成 JSON 字符串。

### 2. 性能替代技术

- **反射对象缓存**：必须用反射时，提前把 `Field`、`Method` 缓存起来，别每次都查。
- **MethodHandle / VarHandle**：Java 7/9 引入的底层 API，直接模拟字节码级别的指令调用。注意 JDK 18 之后核心反射内部也换成了 MethodHandle，两者的性能差距已不像早期那样悬殊，但句柄在类型严格性和访问控制上仍更干净。
- **ASM / ByteBuddy**：不走反射，直接在运行时动态生成字节码文件加载进内存。
- **编译期注解处理（APT）**：像 Lombok、Dagger 这种，在编译期就把代码生成好，运行时完全没有反射损耗。
