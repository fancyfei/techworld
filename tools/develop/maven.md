# 项目自动构建工具-Maven了解

Java世界中主要有三大构建工具：Ant、Maven和Gradle。Ant几乎销声匿迹，Maven应用得最广泛，而Gradle开始快速的发展。

> Maven作用

- Maven给开发人员提供了一个框架，一个完整的生命周期。
- 使用标准的目录结构和默认构建生命周期。
- 基于项目对象模型（POM）的概念。
- Maven的主要功能，分别是依赖管理系统、多模块构建、一致的项目结构、一致的构建模型和插件机制。
- 理念：约定大于配置。
- Maven基本上是Java构建的事实标准。

> Maven概念

1. 本地资源库：本地存储项目的依赖库，默认的文件夹是用户下 “.m2” 目录。
2. 中央存储库：Maven用来下载所有项目的依赖库的默认位置。
3. 私服：特殊的远程仓库，代理广域网上的远程仓库，内网使用，部署自己的构件或者三方的构件。
4. 项目对象模型（POM）：pom.xml，里面声明结构和内容。是整个Maven系统的基本单元。
5. “坐标”：项目组(groupId)，它的名字(artifactId)和版本(version)。保证项目唯一性。

> POM 项目对象模型

1. 包括依赖，配置信息，目标和插件等。每个项目一个。
2. 指定项目本身的坐标，groupId 是项目组的编号，artifactId	这是项目的ID，通常是项目的名称。version 是项目的版本。
3. 超级POM，是Maven的默认POM，指定了一些默认配置和插件，其中的元素可以被覆盖。另外，Maven会有很多原型插件，自带完备的pom.xml。
4. Parent 继承，可以使得子POM可以获得 parent 中的各项配置（依赖插件等）。可用relativePath指定查找位置。它主要是抽取统一的配置信息和依赖版本控制。
5. 聚合（或多模块），在modules中把各模块来聚合成一组项目进行构建，模块名是这些项目的相对目录。SOA和微服务的通用的分解模块方式。

> 主要的构建生命周期

1. *Maven特意设置了标准的项目构建周期，默认情况下所有项目活动的表现上是一致的。*
2. resources：资源复制。这阶段通常是复制静态资源文件。这个未指明的阶段在process-resources包含的。
3. compile：编译，源代码编译在此阶段完成。
4. package：打包，创建JAR/WAR包。
5. install：安装，这一阶段在本地仓库安装程序包。
6. deploy：部署，在远程（私服）仓库安装程序包。
7. clean：可选，清理阶段。

> 项目结构约定
```
src
...main   主体
... ...java 
... ...resources 资源等
... ...webapp    WEB-INF,js,css等
...test   测试类和资源
... ...java
... ...resources
target  构建后所有文件和目录
pom.xml
```
注：在SpringBoot中，webapp里面全部放在resources下面去了。
如下：
```
resources
... application.properties   各种配置文件
... static         js/css/image
... template       页面部分
```
![maven-logo](maven-logo.png)