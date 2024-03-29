# 对开发框架掌握能力要求

## SpringBoot框架
一栈式解决方案。比较轻量，也是当前微服务首选基础框架；去掉了spring繁琐的配置，简化了原有spring开发的流程，提供了各种实用的特性如metric，actuctor等等。

> **入门**

1. 了解SpringBoot框架
2. 运行基于SpringBoot框架的项目
3. 在现有的SpringBoot项目中实现CRUD
4. 了解配置类及@Configuration，能配置Bean

> **掌握**

1. 熟悉SpringBoot框架的组成
2. 独立配置Web项目，包括SpringMVC、Thymeleaf、JPA/Mybatis、内嵌Tomcat等
3. 熟练使用@Configuration及@Bean相关用法
4. 认识并调整各种运行参数，如JVM内存大小，端口等
3. 熟悉各种常见的Starter，如spring-boot-starter-web，spring-boot-starter-test，spring-boot-starter-data-jpa/mybatis-spring-boot-starter等
5. 能写单元测试


> **熟练**

1. 了解SpringBoot项目的打包过程，可以打War和Jar格式
2. 了解监控及其参数，及API文档组件，并能熟练使用
3. 熟悉JavaConfig功能
4. 掌握AutoConfiguration
5. 搭建完整生产级的SpringBoot项目，完整的开发、部署、运维流程
6. 能组合FreeMarker、JSP等前端框架，或者是Vue、Angular等JS框架
7. 可以自定义Starter
8. 熟悉SpringBoot各个组件，并能配合起来，组成解决方案，如安全，spring-data-jpa，cache，spring-data-redis，定时任务，消息，SpringSession等
9. 使用其它Servlet容器
10. 日志配置
11. 了解SpringSecurity或者Shiro

> 注：
>
> Spring开发时的热发布，可以使用JRebel 或者 devtools。
>
> 安全方面，Spring自己也开始推荐Shiro，还是因为它简单吧。
>
> WebSocket方面也是很可能接触到的，需要了解一下。
