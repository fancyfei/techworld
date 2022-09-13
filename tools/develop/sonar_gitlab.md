

## 开启gitlab协同

GitLab上配置：

1，进admin/Applications，新建应用，填写Name，URI。URI一般是sonar地址加：/oauth2/callback/gitlab

2，保存后得到Application ID，Secret。



Sonar上配置：

3，SonarQube，配置，ALM集成页面，选择GitLab。

4，Enable开启，填写GitLab的URL，Application ID，Secret。

5，配置，通用中，填写Server base URL，一般就是sonar自己的地址。



## 配置同步项目的功能

gitlab上的配置：

1，在User Setting中，配置一个access token，要打开api、read_user、read_repository等开关。

2，会获得一个token。



sonar上的配置：

2，配置，ALM集成，GitLab，创建一个GitLab的配置。

3，填写名称，URL，token。URL一般是gitlab地址后面加：/api/v4/



## 分析项目配置

1，在sonar中新建一个token。

2，在GitLab中的具体的项目下，进入配置 > CI/CD > 变量添加下列变量

SONAR_TOKEN，SONAR_HOST_URL

3，项目代码仓库下根目录放文件“.gitlab-ci.yml”。

