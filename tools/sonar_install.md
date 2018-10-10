# sonar安装及配置

# linux服务器上安装sonar
1. 准备环境，安装JDK，注意安装1.8及以上，如果是openjdk，需要devel版本。
2. 下载sonar的zip包，如sonarqube-7.3.zip，然后解压到/usr/local。
3. 修改/config/sonar.properties文件，主要是数据库配置：sonar.jdbc.username=xx sonar.jdbc.password=yy sonar.jdbc.url=zz 都有示例。其他的配置看官方文档。有个参数sonar.web.port是指定sonar web端口号的。
4. 因为不能root用户启动，需要创建一个用户，把sonar所在的目录授权给此用户。注意权限是700，chmod 700 ./sonarqube-7.3
5. 最后一步，启动，bin/linux-x86-64/sonar.sh start，也可以restart重启，stop停止。

# sonar功能的配置使用
- 访问：http://localhost:9000 ，默认用户名及密码是 admin/admin 。
- 安装插件，默认就好。按需求选择git、SonarJava、Chinese Pack 语言包。
- 默认情况下，不用登陆也可以访问，只是不能配置。需要强制登陆，则配置“配置-通用设置-权限”，将Force user authentication勾上。
- 默认情况下，sonar-administrators的用户是不能“执行分析”的。需要在“配置-权限-全局权限”把此项勾上。
- 创建一个普通用户，位置在“配置-权限-用户”，可选在另外一个位置“配置-权限-群组”配置具体的权限。

# 配合Maven使用



# 注意
- 内嵌数据库只能用于测试场景
- sonar不支持mariadb，报错Unsupported mysql version: 5.5. Minimal supported version is 5.6.
- 先配置好数据库，再启动，否则会先使用H2数据库，再迁移，就有可能会报错。
- sonar下载：wget https://sonarsource.bintray.com/Distribution/sonarqube/sonarqube-7.3.zip
