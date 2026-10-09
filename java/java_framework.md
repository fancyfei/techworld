# Java 主要开发框架简述

## 什么是框架

框架（Framework）是一组可复用的通用功能代码，同时规定了应用的体系结构：框架负责提供基础设施并调度流程，开发者按框架的约定填充自己的业务代码。

> 与调用一个工具类库不同，使用框架时主控权在框架一侧，这种现象称为控制反转（IoC），依赖注入（DI）是实现控制反转的主要手段。

## Spring 生态

绕不开的 Spring。Spring 自 2003 年发布以来逐步覆盖了 Java 服务端开发的各个环节，围绕它形成了 Spring Framework、Spring Boot、Spring Cloud 等一系列项目，业内常称其为"Spring 全家桶"。在各种开发者调查中，Spring Boot 长期是使用率最高的 Java Web 框架，招聘市场上的 Java 服务端岗位也普遍以 Spring 技术栈为前提。

## Web 层框架

Web 层框架负责接收 HTTP 请求、组织业务调用并返回响应，主流是 Spring MVC：基于 Servlet 栈的同步阻塞模型，注解驱动，是绝大多数 Java Web 应用的默认选择。同一体系内还有 Spring WebFlux，采用响应式编程模型和非阻塞 I/O，面向高并发、流式处理场景，但学习成本和调试复杂度更高。

> 早期的 Struts2 曾牛逼过，现在已经只存在于遗留系统中。
>

## Spring Boot

传统 Spring 应用 XML 配置，繁琐。

Spring Boot（2014 年发布）用三条约定解决了这个问题：

- 约定优于配置。
- 自动装配根据类路径推断并组装 Bean
- 内嵌 Tomcat 或 Jetty 使应用可以直接以 jar 包运行。

配合按功能拆分的 starter 依赖，新项目基本以 Spring Boot 作为起点。2025 年 11 月发布的 Spring Framework 7 与 Spring Boot 4 将基线提高到 Java 17 和 Jakarta EE 11，并原生支持虚拟线程和 GraalVM 原生镜像。

## 数据访问框架

Java 访问关系型数据库有两条主路线。

一条是 ORM 路线：JPA（Java Persistence API）是官方规范，本身不是实现，最常用的实现是 Hibernate，Spring Data JPA 在其上又封装了仓库接口，绝大多数场景不再需要手写 SQL。

另一条是 SQL 映射路线：MyBatis 把 SQL 语句与 Java 接口方法做映射，SQL 由开发者完全掌控，在复杂查询多、需要精细优化 SQL 的项目中更为常见，国内使用面很广。

简单场景下 Spring 自带的 JdbcTemplate 也足够。

## 前后端分离

前后端分离普及之前，页面由服务端渲染，模板引擎负责把数据填进 HTML：JSP/JSTL 是 Java EE 时代的标准方案，FreeMarker 语法更简洁，Thymeleaf（2011 年发布）以自然的 HTML 标签语法成为 Spring 官方文档的推荐选项。

随着前端工程化成熟，新项目普遍采用后端提供 REST 接口、前端独立开发与部署的架构，服务端模板引擎的方案基本只在遗留系统中。

## 服务化与微服务框架

### SOA

SOA 时代（约 2008–2015 年）的主流做法是用远程调用把单体拆成服务：

- Dubbo 出自阿里巴巴，提供基于接口的 RPC 和服务治理；
- WebService 依托 SOAP 协议和 JAX-WS 规范（Apache CXF 是常用实现）；
- 更重的方案会引入 ESB 企业服务总线（如 Mule）集中编排服务。这套体系在大型企业遗留系统中仍广泛存在。

### 微服务

微服务架构兴起后，Spring Cloud 成为主流方案。它基于 Spring Boot，提供注册中心、配置中心、网关、熔断、负载均衡等分布式能力。

早期的 Netflix 套件（Eureka、Zuul、Ribbon、Hystrix）已进入维护模式，官方与社区分别用 Spring Cloud Gateway、Spring Cloud LoadBalancer、Resilience4j 等替代；在国内，Spring Cloud Alibaba 集成的 Nacos（注册中心兼配置中心）和 Sentinel（流量防护）使用非常普遍。

性能更高的 Dubbo 也在 2018 年进入 Apache 基金会并成为顶级项目，当前主版本为 Dubbo 3，注册中心常用 Nacos，ZooKeeper 也是可选方案之一，新增的 Triple 协议基于 gRPC，便于跨语言调用。

### 云原生

容器与 Serverless 环境对启动速度和内存占用提出了新要求，由此出现了云原生方向的框架：Quarkus（红帽）与 Micronaut 都采用编译期依赖注入，可将应用编译为 GraalVM 原生镜像，冷启动可达毫秒级、内存占用显著低于传统 JVM 模式。

> 它们的社区规模尚不及 Spring，但已是云原生选型时的常见选项。

