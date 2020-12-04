# Nginx主要配置

Nginx配置比较简单，配置文件主要由四部分组成：main(全区设置)，server(主机配置)，upstream(负载均衡服务器设置)，和location(URL匹配特定位置设置)。加上新版的Nginx支持四层负载均衡的stream（四层代理配置）。

Nginx的主配置文件是conf文件夹的nginx.conf文件，其中虚拟主机部分可以拆分到conf.d目录下的多个文件中，然后在主配置文件中include加载上。

## 配置结构如下：

```yaml
...              #全局块
events {         #events块
   ...
}
http      #http块
{
    ...   #http全局块
    upstream name       #upstream块
    { 
        server ...       #被代理服务器地址，以及参数
    }
    server        #server块
    { 
        ...       #server全局块
        location [PATTERN]   #location块
        {
            ...
        }
        location [PATTERN] {...}
    }
    server{...}
    ...     #http全局块
}
stream {...}      #stream块，四层负载均衡
```

- **全局块**：配置影响nginx全局的指令。一般有运行nginx服务器的用户组，nginx进程pid存放路径，日志存放路径，配置文件引入，允许生成worker process数等。
- **events块**：配置影响nginx服务器或与用户的网络连接。有每个进程的最大连接数，选取哪种事件驱动模型处理连接请求，是否允许同时接受多个网路连接，开启多个网络连接序列化等。
- **http块**：可以嵌套多个server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。如文件引入，mime-type定义，日志自定义，是否使用sendfile传输文件，连接超时时间，单连接请求数等。
- **server块**：配置虚拟主机的相关参数，一个http中可以有多个server。
- **upstream块**：指令主要用于负载均衡，设置一系列的后端服务器。
- **location块**：配置请求的路由，以及各种页面的处理情况。
- **stream块**：ngx_stream_core_module模块，使nginx支持四层负载均衡，内部主要有upstream与server配置项。

## 配置常见参数

- $remote_addr 与 $http_x_forwarded_for 用以记录客户端的ip地址；
- $remote_user ：用来记录客户端用户名称；
- $time_local ： 用来记录访问时间与时区；
- $request ： 用来记录请求的url与http协议；
- $status ： 用来记录请求状态；成功是200；
- $body_bytes_s ent ：记录发送给客户端文件主体内容大小；
- $http_referer ：用来记录从那个页面链接访问过来的；
- $http_user_agent ：记录客户端浏览器的相关信息；
- $http_host：请求地址，即浏览器中输入的地址（IP或域名）；
- $request_uri：请求参数的原始URI，不可修改；
- $uri：请求中的当前URI(不带请求参数，参数位于$args)，$uri不包含主机名，可修改；
- $args和$query_string：请求中的参数值；

## Nginx部署命令

- 验证配置是否正确: nginx -t
- 查看Nginx的版本号：nginx -V
- 启动Nginx：start nginx
- 快速停止或关闭Nginx：nginx -s stop
- 正常停止或关闭Nginx：nginx -s quit
- 配置文件修改重装载命令：nginx -s reload
- 系统自启动：systemctl enable nginx

