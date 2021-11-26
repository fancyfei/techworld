# 对开发框架掌握能力要求

## SOA及微服务
- SOA通常Rpc用WebService，框架用ESB。
- SOA总线的轻量方案Nginx+hosts/dns，可配合各种RPC。
- SOA最常用的框架应当还是Dubbo，Rpc用Hessian2。
- Dubbo同时也可以是实现微服务的框架之一，微服务应用最广泛的还是SpringCloud那一家子。
- 微服务需要配置容器才能“微”，这基本上是Docker的世界了。而容器的运维被kubernetes（K8s）接管了，Swarm渐渐式微。
- 微服务下一代？Service Mesh（服务网格）也许准备好了。

---
> **入门**

1. 了解SOA及微服务的概念
2. 了解WebService，编写简单的实现
3. 了解ESB
4. 了解Dubbo
5. 了解SpringCloud
6. 了解异步模式，了解消息机制，了解缓存的使用

> **掌握**

1. 使用Jax-WS或者CXF，编写WebService接口，实现客户端调用
2. 准备简单的Dubbo开发环境，发布服务，调用服务
3. 准备简单的SpringCloud开发环境，结合Eureka、ZUUL、Feign，实现多个微服务
4. 在系统中熟练使用缓存
5. 在系统中熟练应用消息
6. 了解Zookeeper或Eureka等服务注册组件
7. 了解负载均衡
8. 了解分布式，集群
9. 简单配置ZUUL，Nginx

> **熟练**

1. 可以搭建一个基于Dubbo的或者SpringCloud的系统框架
2. 熟练使用SpringCloud各个组件，掌握各组件的应用场景及原理
3. 能熟练使用Feign或RestTemplate进行RPC接口调用
3. Dubbo配合使用Zookeeper、Nginx、Consul等
4. SpringCloud中配置Eureka、Zuul/Gateway等
5. 配置Redis、RabbitMQ等
6. 能解决分布式Session问题，分布式事务问题，RPC接口调用问题
7. 可以集群部署，可以编写运维脚本
8. 能使用日志和监控分析和解决遇到的问题

> **注**

- Dubbo有一段时间停止更新了，阿里内部也不怎么使用了。2017年后，又开始更新了。
- Dubbo被当当网拿来支持rest后，成为DubboX，也是广泛使用的一个版本。
- SpringCloud组件里面Netflix提供了几个重要的组件，但是更新到2.0后，逐渐的脱离Netflix。