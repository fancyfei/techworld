# Spring Resource 典型用法

Spring 用三个接口覆盖资源的定位与读取：`Resource`、 `ResourceLoader`、 `ResourcePatternResolver`。

## 读取单个资源

### 1. 三种来源的构造方式

| 来源 | 构造方式 | 典型场景 |
|------|----------|----------|
| classpath | `new ClassPathResource("sql/query-user.sql")` | 应用内置的 SQL、JSON、模板、证书 |
| 文件系统 | `new FileSystemResource("/opt/myapp/config/rules.json")` | 部署目录中需独立更新的配置、规则、模板 |
| 远程 URL | `resourceLoader.getResource("https://example.com/config.json")` | 通过 HTTP/HTTPS 获取远程文件 |

三种来源的读取方式统一：

```java
try (InputStream input = resource.getInputStream()) {
    String content = new String(input.readAllBytes(), StandardCharsets.UTF_8);
}
```

classpath 资源的目录结构示例：

```text
src/main/resources/
├── sql/query-user.sql
├── config/rules.json
└── templates/notice.html
```

注意：

- classpath 路径相对于 classpath 根目录，不需要添加 `src/main/resources` 前缀。
- 读取资源使用 `getInputStream()`，不要默认使用 `getFile()`。应用打包成 JAR 后，classpath 资源位于 JAR 内部，不对应独立的操作系统文件。
- 外部文件可以独立于应用包更新，但需要自行处理文件权限、文件缺失及内容刷新。
- 远程资源的读取受网络、超时和服务器状态影响，生产环境应明确超时、重试和异常处理策略；需要精细控制 HTTP 请求行为时，主流做法是使用专门的 HTTP 客户端，而不是 `UrlResource`。

### 2. 通过配置指定资源位置

适用于开发、测试、生产环境使用不同资源路径的场景。

`application.yml`：

```yaml
app:
  template:
    location: classpath:templates/notice.html
```

注入：

```java
@Value("${app.template.location}")
private Resource template;
```

生产环境改为：

```yaml
app:
  template:
    location: file:/opt/myapp/templates/notice.html
```

`@Value` 注入的字符串会经 `ResourceLoader` 解析为对应的 `Resource` 实现，业务代码不需要区分资源来自 classpath 还是文件系统。

资源配置较多时，主流做法是使用 `@ConfigurationProperties` 将多个 `Resource` 属性集中管理，而不是在不同业务类中重复声明 `@Value`。

## 批量加载资源

### 1. 编程式扫描

适用于批量加载 SQL 文件、规则文件、插件配置或模板。

```java
ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();

Resource[] resources = resolver.getResources("classpath*:sql/*.sql");

for (Resource resource : resources) {
    // 读取并注册规则
}
```

常见路径表达式：

| 表达式 | 用途 |
|--------|------|
| `classpath:sql/query.sql` | 读取指定资源 |
| `classpath*:sql/*.sql` | 搜索多个 classpath 位置中的 SQL 文件 |
| `classpath*:sql/**/*.sql` | 搜索 SQL 目录及其子目录中的文件 |

主应用与依赖 JAR 都包含规则文件时，使用 `classpath*:` 才能搜到所有 classpath 位置。

### 2. 声明式注入

匹配规则固定时，可以直接注入资源数组：

```java
@Value("classpath*:rules/*.json")
private Resource[] ruleResources;
```

资源路径需要根据运行参数动态生成时，改用 `ResourcePatternResolver`。

注意：不要依赖扫描结果的默认顺序。加载顺序影响业务结果时应显式排序；存在同名资源时应明确冲突处理规则。

## 在业务逻辑中使用资源内容

### 1. 通用模式：读取一次并缓存

SQL 文件、模板文件、初始化数据的用法相同：读取一次内容，缓存为字符串或解析后的对象，交给业务使用。通用的读取辅助方法：

```java
private String readResource(Resource resource) throws IOException {
    try (InputStream input = resource.getInputStream()) {
        return new String(input.readAllBytes(), StandardCharsets.UTF_8);
    }
}
```

### 2. 场景一：SQL 文件

SQL 较长、需要单独维护或多个业务逻辑复用时，独立存放在资源文件中。

`sql/query-user.sql`：

```sql
SELECT id, username, status
FROM sys_user
WHERE status = ?
```

读取并执行：

```java
String sql = readResource(new ClassPathResource("sql/query-user.sql"));
jdbcTemplate.query(sql, rowMapper, status);
```

执行时继续使用参数绑定。启动后不会变化的 SQL，在 Bean 初始化阶段读取一次并缓存，避免每次数据库操作都读文件；运行期间允许更新的 SQL，需要单独设计刷新机制。

### 3. 场景二：模板文件

邮件、短信、通知、导出等模板：

```java
@Value("${app.template.location:classpath:templates/notice.html}")
private Resource template;
String content = readResource(template);
```

`${...:默认值}` 中冒号后是默认位置，配置项缺省时生效。模板变量替换、条件判断和循环渲染属于模板处理逻辑，不是 `Resource` 的职责：简单文本使用字符串替换，复杂模板使用专门的模板引擎。模板需要由运维人员独立更新时，放在外部目录并通过配置指定位置（配置方式见"通过配置指定资源位置"）。

### 4. 场景三：启动初始化数据

应用启动时读取初始化规则、字典文件、映射关系或静态元数据：

```java
@PostConstruct
public void init() throws IOException {
    Resource resource = new ClassPathResource("config/rules.json");
    // 读取、解析、校验后注册到内存
}
```

必须存在的资源，在初始化阶段完成读取和内容校验；资源缺失或内容不合法时直接报错，避免应用启动成功后才在业务请求中暴露问题。

`@PostConstruct` 适合简单初始化。初始化依赖其他 Bean 的完整生命周期或需要控制启动失败行为时，常见替代机制有 `ApplicationRunner`、监听 `ApplicationReadyEvent`、实现 `InitializingBean`。

## 资源作为 Web 下载响应

### 1. 磁盘或 classpath 文件下载

适用于 Spring MVC 接口返回 PDF、Excel、模板文件等：

```java
Resource resource = new ClassPathResource("files/manual.pdf");

return ResponseEntity.ok()
        .header(
                HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=\"manual.pdf\""
        )
        .contentType(MediaType.APPLICATION_PDF)
        .body(resource);
```

外部文件将资源实现替换为 `FileSystemResource` 即可。注意：

- 文件名包含中文等非 ASCII 字符时，直接拼接 `Content-Disposition` 会乱码。主流做法是使用 `ContentDisposition.builder().filename(name, StandardCharsets.UTF_8).build()` 生成符合 RFC 5987 的编码值。
- 用户指定文件路径的下载接口必须进行权限校验和路径限制，不能直接拼接请求参数后读取任意文件。

### 2. 内存字节数据

适用于动态生成的 CSV、体积较小的导出文件或已有字节数组：

```java
byte[] bytes = generateCsv();

return ResponseEntity.ok()
        .header(
                HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=\"data.csv\""
        )
        .contentLength(bytes.length)
        .body(new ByteArrayResource(bytes));
```

`ByteArrayResource` 适合数据已经位于内存中的场景。文件体积较大时应改为流式输出，避免将整个文件加载到 `byte[]` 中造成额外内存压力。

### 3. 已打开的输入流

底层组件已经提供 `InputStream`，需要作为下载响应时：

```java
long length = file.length();
Resource resource = new InputStreamResource(new FileInputStream(file));

return ResponseEntity.ok()
        .contentLength(length)
        .body(resource);
```

`InputStreamResource` 包装的流只能读取一次，其 `contentLength()` 的默认实现会把整个流读完才能算出长度。用作响应体时，不要在 `InputStreamResource` 上调用 `contentLength()`；长度应从外部信息（如 `file.length()`）获取，并通过 `.contentLength(...)` 显式设置。否则写入响应体时会抛出 `IllegalStateException: InputStream has already been read`。

## 读取的通用注意事项

### 1. 存在性与可读性检查

```java
if (!resource.exists()) {
    throw new IllegalStateException("资源不存在");
}

if (!resource.isReadable()) {
    throw new IllegalStateException("资源不可读");
}
```

这类检查适合启动校验和错误提示，但不能保证后续读取一定成功：文件可能在检查后被删除，远程资源也可能在读取时发生故障。必须存在的资源，在初始化阶段直接完成实际读取和内容校验更可靠。

### 2. 输入流的关闭

读取资源时通过 try-with-resources 关闭输入流。注意区分两个生命周期：`Resource` 用于描述和访问资源，可以长期持有；`InputStream` 是单次读取操作中需要管理的流对象，用完即关。

### 3. 文件名与描述信息

```java
String filename = resource.getFilename();
String description = resource.getDescription();
```

- `getFilename()`：获取文件名，用于日志和文件名展示。某些资源没有明确文件名，返回值可能为 `null`。
- `getDescription()`：获取资源描述，用于异常信息和问题定位。

两者都不应被当作资源内容。