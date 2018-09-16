# Java最基础也最流行的开发框架简述
> 认识框架（Framework）：

1. 一组抽象构件及构件实例间交互的方法。
2. 是通用完备功能（即基础设施）的代码重用。
3. 同时也是应用开发者定制的应用的体系结构。
4. 也表现为特定领域的复用的组件、通用功能的模板化、统一的规范。
5. *Spring一统天下*

---
> 在Web应用的场景下，Java开发流行且需要重点掌握的开发框架。

1. Spring全家桶，包括简化配置而生的SpringBoot。

2. 数据库访问的框架Jpa和Mybatis，同时Jpa的具体实现通常是Hibernate。

3. MVC框架除了SpringMVC，也常用Struts2。

4. 微服务，主要的框架是Dubbo和SpringCloud。

5. 页面视图框架，常用Jsp/JSTL，FreeMarker，和新推出的Thymeleaf等。

6. SOA最多用的应当是Dubbo和WebService。

---
> 顺带说说SOA的解决方案
- *SOA的RPC，有大一统的框架Spring Remote，可集成Hessian、JAX-WS等。WebService的框架常用JAX-WS，框架cxf，axis2也比较流行等*
- *SOA总线的方案比较轻量级的方案是结合Nginx+hosts/dns，配合各种RPC。也有直接使用ESB的，如Mule。*
- *Dubbo也SOA的常用方案，他多数是和Zookeeper一起用的。*

---
> 另外由于前端的重要性越来越高，Java开发人员也常常会使用其中一些框架。
- 前端JS的框架就比较多样化了，常用的Jquery及其各种插件，Ext等，前后端分离的框架有angularjs，React，VUE等，同时ES6也已经流行。
- 顺便提一下，CSS相关也需关注流行的Sass和Less等。


