# Nginx的关于性能配置项

Nginx之所以性能如此优越，是因为充分利用操作系统的内核。提供了异步的、非阻塞的Web服务，有很多的模块，包括第三方模块。比较显著的性能提升在系统资源的消耗较低，对多核的充分利用，对网络IO进行优化。

## 事件模块与处理进程

```yaml
events {
	use epoll;
	worker_connections 204800;
	keepalive_timeout 60;
}
```

- use首选epoll，Nginx支持的工作模式有select、poll、kqueue、epoll、rtsig和/dev/poll。其中select和poll都是标准的工作模式，kqueue和epoll是高效的工作模式，不同的是epoll用在Linux平台上，而kqueue用在BSD系统中。
- worker_connections，每个进程的最大连接数，默认1024；最大客户端连接数max_clients = worker_processes * worker_connections/4（需要打开Linux系统进程的最大打开文件数限制“ulimit -n 65536”）。

## 全局多核利用

```yaml
worker_processes 8;
worker_rlimit_nofile 204800;
```

- worker_processes，工作进程个数，一般设置cpu的核心或者核心数x2。
- worker_rlimit_nofile，一个进程可打开的最多文件描述符数目，理论总数应当是最多打开文件数（ulimit -n），最好与ulimit -n的值保持一致。操作系统配置是/etc/security/limits.conf：“soft  nofile  65535”、“soft  nofile  65535”。

## HTTP高效传输

```yaml
http {
    sendfile on;
    keepalive_timeout 60;
    send_timeout 10;
    client_max_body_size 10m;
    ......
}
```

- sendfile 可以让sendfile()发挥作用。开启可以在内核中完成数据的copy，跳过用户空间中多次复制。
- keepalive_timeout 给客户端分配keep-alive链接超时时间，设置低些可以及时断开无用连接。
- send_timeout 指定响应客户端的超时时间。是两个连接活动之间的时间，如果客户端没有任何活动，关闭连接。
- client_max_body_size 允许上传文件的大小。

## gzip调优

```yaml
http {
    gzip on;
    gzip_buffers 4 16k;
    gzip_min_length 1000; 
    gzip_comp_level 4; 
    gzip_types text/plain text/css;
    ......
}
```

- gzip用于设置开启或者关闭gzip模块，“gzip on”表示开启GZIP压缩，实时压缩输出数据流；
- gzip_buffers申请多个少块多少内存做结果流缓存。
- gzip_min_length设置允许压缩的页面最小字节数，过小的文件压缩后可能变大。
- gzip_comp_level用来指定GZIP压缩比，数值越小效果也差，处理越快。
- gzip_types用来指定压缩的类型，“text/html”类型总是会被压缩的。

## HTTP代理的缓存

```yaml
http {
    proxy_cache_path /path/to/cache levels=1:2 keys_zone=mycache:10m max_size=10g inactive=60m
    proxy_buffering on;
    proxy_buffers 4 32k
    proxy_buffer_size 4k;    
    client_header_buffer_size 4k;
    large_client_header_buffers 4 8k;
    proxy_cache mycache;
    proxy_no_cache $cookie_nocache $arg_nocache;
    server {
        location ~ .*\.(htm|html)$ {
            expires 24h;
            root  /opt/app/code;
        }
    }
}
```

- proxy_cache_path，指定用于缓存的本地磁盘目录是 /path/to/cache/。levels多级目录；keys_zone共享内存区大小；max_size缓存的上限；inactive缓存保持时间。
- proxy_buffering，开启或关闭。
- proxy_buffers，缓冲区，单个连接缓存来自后端realserver的response。
- proxy_buffer_size，指定后端 response 的 buffer 的大小。
- client_header_buffer_size 客户端请求头部的缓冲区大小。超过后会以large_client_header_buffers配置，请求行仍然过大则返回414错误。请求头过大则返回400错误。
- proxy_cache，指定共享内存区启用proxy_cache。默认是off关闭。它需要设置proxy_buffering on;
- proxy_no_cache，对哪些条件不启用proxy_cache，后面跟的参数是变量。和proxy_cache_bypass类似。
- location中的expires，在location或if段里配置浏览器端的过期时间。客户端发送带etag、if-modified-since头部信息的请求，Nginx验证请求的文档在本地缓存中是否过期，文档没有过期返回304状态码给浏览器。
- 注1：缓存中还涉及两个额外的NGINX进程：**cache manager** 周期性地启动，检查高速缓存的状态。清理缓存目录中的数据。**cache loader** NGINX 启动时运行一次，将之前的缓存加载到共享内存区域。
- 注2：proxy模块的缓存配置可同时作用于http、server、location。