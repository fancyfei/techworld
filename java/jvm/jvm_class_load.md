# JVM 类加载机制

类加载器（ClassLoader）负责把 class 文件加载进 JVM。JVM 内置三个类加载器：启动类加载器（Bootstrap ClassLoader）、扩展类加载器（Extension ClassLoader）、应用程序类加载器（Application ClassLoader）。三者构成父子层级：应用程序类加载器的父加载器是扩展类加载器，扩展类加载器的父加载器是启动类加载器。启动类加载器由 JVM 原生代码实现，不是 Java 类，程序无法直接获取和操纵；另外两个加载器都是 `java.lang.ClassLoader` 的子类。用户自定义的类加载器一般直接继承 `java.lang.ClassLoader`。

JVM 对 class 文件按需加载：一个类在第一次被主动使用（如创建实例、访问静态成员）时才触发加载。

## 三个内置类加载器

- 启动类加载器：加载 JDK 核心类库（如 `java.lang.*`），搜索路径可由 `-Xbootclasspath` 指定。
- 扩展类加载器：加载 `<JAVA_HOME>/lib/ext` 目录或 `java.ext.dirs` 指定的扩展类库。JDK 9 起它被平台类加载器（Platform ClassLoader）取代。
- 应用程序类加载器：加载 classpath 上的应用类，是程序运行时的默认类加载器。

## 类加载流程

类加载的入口是 `java.lang.ClassLoader` 的 `loadClass` 方法，默认流程如下：

1. 调用 `findLoadedClass` 检查该类是否已加载，已加载则直接返回对应的 `Class` 对象。
2. 缓存未命中，把加载请求委托给父加载器；父加载器继续向上委托，直到启动类加载器。
3. 父加载器加载失败，才轮到当前加载器自己加载：读取 class 文件的二进制数据，交给 `defineClass` 转换为 `Class` 对象；文件找不到或转换失败，抛出 `ClassNotFoundException`。

这条流程体现了类加载机制的三个特点：

- 缓存机制：加载过的类会被缓存，重复加载时直接从缓存返回。
- 双亲委派：加载请求先沿父加载器链向上传递，失败后才自己加载，详见下一节。
- 全盘负责：一个类加载器加载某个类时，该类依赖和引用的其他类也由同一个加载器载入。

## 双亲委派机制

双亲委派的含义：收到加载请求时先委托父加载器处理，父加载器加载不到才自己加载。"双亲"是 parent delegation 的翻译遗留，实际指父加载器链条。整个过程先自底向上检查类是否已加载，再自顶向下尝试加载：

- 类加载器收到请求，先把请求委托给父加载器。
- 父加载器还有父加载器则继续向上，直到启动类加载器。
- 父加载器能完成加载就返回结果；加载失败则由子加载器自己尝试，仍失败就抛出 `ClassNotFoundException`。

双亲委派保证 Java 核心库的类型安全：用户代码无法定义一个全限定名与核心类相同的类来覆盖它；同一个类也只会被一个加载器加载，避免内存中出现多份相同的字节码。

## 自定义类加载器

自定义类加载器继承 `java.lang.ClassLoader`。`loadClass` 是加载入口，内部依次调用 `findLoadedClass`、父加载器委托和 `findClass`。

- 不打破双亲委派：只重写 `findClass`，在其中读取 .class 文件，把二进制数据交给 `defineClass` 转换为 `Class` 实例。
- 打破双亲委派：重写整个 `loadClass`。

两个典型的打破场景：

- Tomcat：为每个 Web 应用提供独立的 WebappClassLoader，应用类优先由自己加载，以隔离各应用的 class；`common` 目录下的类则由上层 CommonClassLoader 共享。
- SPI：JDBC 这类技术标准只由 Java 核心库提供接口，厂商实现类放在 classpath 上。核心类库由启动类加载器加载，需要加载接口的实现类时，父加载器无法请求子加载器完成，只能借助线程上下文类加载器（Thread Context ClassLoader，默认为应用程序类加载器）来加载实现类。
