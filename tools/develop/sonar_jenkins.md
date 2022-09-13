# SonarQube与Jenkins集成





1，Jenkins->插件管理，安装插件：SonarQube Scanner for Jenkins

2，SonarQube菜单：My Account -> Security，生成token。

3，生成Jenkins的全局凭据，填写上面的token。

4，jenkins -> 系统设置，配置Sonar Sever。

5，全局工具配置，配置SonarQube Scanner。



6，新建Job，正常的配置代码仓库。

7，Job中，SonarQube Scanner参数需要配置：

> sonar.projectKey=xxx
> sonar.projectName=x
> sonar.projectVersion=x.x
> sonar.sources=src
> sonar.java.binaries=target/classes
> sonar.language=java

8，在SonarQube创建同名项目。

9，完成，运行。