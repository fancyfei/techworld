# Spring 资源管理设计思想和架构

## 一、核心思想与设计理念

Spring 资源管理的核心是统一资源抽象，将类路径、文件系统、URL 等不同来源的资源统一为 `Resource` 接口，屏蔽底层访问方式的差异。

核心职责分为三个层次：

- `Resource`：描述资源及其访问能力。
- `ResourceLoader`：根据资源位置定位资源。
- `ResourcePatternResolver`：根据资源模式查找多个资源。

资源定位与资源读取相互分离。获取 `Resource` 对象不代表资源内容已经读取，实际读取通常发生在调用 `getInputStream()` 时。

```
Resource resource = resourceLoader.getResource(
        "classpath:templates/order.html");

try (InputStream in = resource.getInputStream()) {
    // 在需要时读取资源内容
}
```

这种设计使资源存储位置与业务逻辑解耦。资源从文件系统迁移到类路径或 JAR 包内部时，上层代码不必直接依赖具体的文件访问 API。

## 二、底层架构与接口设计

### 1. InputStreamSource：统一数据读取入口

```
public interface InputStreamSource {
    InputStream getInputStream() throws IOException;
}
```

定义统一的输入流获取方式，是 Spring 资源读取抽象的基础。

### 2. Resource：资源访问抽象

`Resource` 继承 `InputStreamSource`，在输入流访问能力之上，扩展资源状态、定位信息和元数据访问能力。

| 核心方法               | 作用                   |
| ---------------------- | ---------------------- |
| `getInputStream()`     | 获取资源输入流         |
| `exists()`             | 判断资源是否存在       |
| `isReadable()`         | 判断资源是否可读       |
| `getURL()`、`getURI()` | 获取资源定位地址       |
| `getFile()`            | 尝试获取对应的文件对象 |
| `contentLength()`      | 获取资源内容长度       |
| `getFilename()`        | 获取资源名称           |
| `getDescription()`     | 获取资源描述信息       |

需要注意，`getFile()` 不是通用的资源访问方式。JAR 内部的资源可以通过输入流读取，却不一定对应独立的磁盘文件。

**实践要点：** 通用资源处理优先使用 `getInputStream()`，不要默认所有资源都能转换成 `File`。

### 3. ResourceLoader：资源定位

```
public interface ResourceLoader {
    Resource getResource(String location);
    ClassLoader getClassLoader();
}
```

常见资源位置包括：

```
classpath:config/app.properties
file:/opt/app/config/app.properties
https://example.com/config.properties
```

没有显式前缀时，资源如何定位取决于具体的加载器实现。

### 4. ResourcePatternResolver：批量资源解析

```
public interface ResourcePatternResolver
        extends ResourceLoader {

    Resource[] getResources(String locationPattern)
            throws IOException;
}
```

它在单个资源定位的基础上增加通配符匹配和多资源查找能力。

- `classpath:`：定位类路径资源。
- `classpath*:`：查找类路径中所有匹配的资源。
- `*.properties`：匹配指定目录下的属性文件。
- `**/*.xml`：匹配多个层级目录下的 XML 文件。

例如：

```
ResourcePatternResolver resolver =
        new PathMatchingResourcePatternResolver();

Resource[] resources = resolver.getResources(
        "classpath*:META-INF/*.properties");
```

该方式可以搜索多个 JAR 中符合条件的资源，适用于框架扩展配置、插件发现和多模块资源加载。

核心架构关系：

```
InputStreamSource
        ↑
     Resource

ResourceLoader
        ↑
ResourcePatternResolver
```

前者定义资源读取能力，后者定义资源定位及批量匹配能力。

## 三、核心实现类体系

### 1. Resource 实现类

| 实现类                   | 适用场景                             |
| ------------------------ | ------------------------------------ |
| `ClassPathResource`      | 类路径资源，包括 JAR 内部资源        |
| `FileSystemResource`     | 文件系统资源                         |
| `PathResource`           | 基于 NIO `Path` 的资源               |
| `UrlResource`            | URL 定位的资源                       |
| `ServletContextResource` | Servlet 应用上下文资源               |
| `InputStreamResource`    | 已有输入流的封装，通常不适合重复读取 |
| `ByteArrayResource`      | 内存字节数组资源                     |

不同实现类只负责适配各自的底层资源类型，对外统一提供 `Resource` 接口。

### 2. ResourceLoader 实现类

| 实现类                                | 核心职责                                  |
| ------------------------------------- | ----------------------------------------- |
| `DefaultResourceLoader`               | 默认资源定位，支持类路径和 URL 等资源形式 |
| `FileSystemResourceLoader`            | 将普通路径按文件系统资源解析              |
| `PathMatchingResourcePatternResolver` | 处理通配符、`classpath*:` 和批量资源查找  |

`PathMatchingResourcePatternResolver` 通常组合一个 `ResourceLoader` 使用：前者负责模式匹配，后者负责实际资源定位。

## 四、与 Spring 容器的集成

### 1. ApplicationContext：统一资源入口

`ApplicationContext` 继承了 `ResourcePatternResolver`，本身具备单个资源定位和批量资源查找能力。

```
Resource resource = applicationContext.getResource(
        "classpath:config/app.properties");

Resource[] resources = applicationContext.getResources(
        "classpath*:META-INF/*.properties");
```

因此，Spring 容器不仅管理 Bean，也提供统一的资源加载能力。

### 2. BeanDefinition 加载

XML 配置通过资源抽象读取，再由 `XmlBeanDefinitionReader` 解析并注册 Bean 定义。

```
资源位置
   ↓
Resource
   ↓
XmlBeanDefinitionReader
   ↓
解析 XML
   ↓
BeanDefinitionRegistry
   ↓
BeanFactory 创建 Bean
```

资源层负责提供配置内容，读取器负责解析，注册表负责保存 Bean 定义。配置文件位于文件系统还是 JAR 包中，不改变后续的 Bean 定义处理流程。

### 3. 组件扫描

组件扫描同样依赖资源模式解析机制。

```
扫描包
   ↓
ResourcePatternResolver
   ↓
查找 .class 资源
   ↓
MetadataReader 读取类元数据
   ↓
判断扫描条件
   ↓
注册 BeanDefinition
```

Spring 不需要只依靠文件系统遍历查找类文件，也可以处理 JAR 内部的类路径资源。

### 4. ResourceLoaderAware：向 Bean 提供资源加载能力

```
public interface ResourceLoaderAware {
    void setResourceLoader(ResourceLoader resourceLoader);
}
```

实现该接口的 Bean 可以获取容器提供的 `ResourceLoader`，从而动态定位资源。

在普通业务代码中，使用构造器注入通常更直接：

```
@Service
public class TemplateService {

    private final ResourceLoader resourceLoader;

    public TemplateService(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }

    public Resource load(String location) {
        return resourceLoader.getResource(location);
    }
}
```

需要动态加载模板、规则文件或外置配置时，这种方式可以避免业务代码直接依赖整个 `ApplicationContext`。

