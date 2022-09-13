# 软件开发之持续集成-jenkins安装配置

# 在linux安装
- 环境准备，主要安装Java环境，略过。
- 安装jenkins，最简单的是yum安装，略过。主要说war包安装，其实也就下载war包，将war包放到/usr/local/jenkins下，就可以 java -jar xx.war启动了。最好nohup后台启动。后面可以跟上内存大小，端口号等参数。
- 打开web界面，默认是8080端口。需要首次密码，这个密码在~/.jenkins/secrets/initialAdminPassword文件中。
- 正常进入后，安装插件，可以默认。手工选择的话，至少把git，maven带上。
- 设置管理员的用户名密码。
- 经过非常简单的一个过程，就可以正常登陆了。

# 配置git
- git的安装，直接yum就行。
- 先在jenkins服务器上root用户生成密钥对。ssh-keygen -t rsa
- 配置和仓库的关系，将公钥文件.ssh/id_rsa.pub内容复制到private deploy key中。
- 配置和jenkins的关系，将私钥文件.ssh/id_rsa的内容复制到jenkins的private key中去。
- 这两个关系配置好，就可以正常的从仓库拉代码了。

# 配置maven
- 如果选择自动安装maven，则jenkins中配置中的maven章节选中“自动安装”即可。