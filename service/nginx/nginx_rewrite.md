# Nginx的rewrite和location

Nginx的rewrite和location的功能都能实现跳转，主要区别在于rewrite常用于server内更改获取资源的路径，而location是对一类路径做控制访问和反向代理，可以proxy_pass到其他服务器。Nginx通过ngx_http_rewrite_module模块支持url重写、支持if条件判断，但不支持else。默认已经安装该模块，需要在安装时加上PCRE支持。

结合Nginx的变量、正则表达式、标志位实现url重写以及重定向。rewrite生效位置是server{},location{},if{}中，并且只能对域名后边的document_uri起作用。

## Location

```yaml
location ~* /js/.*/\.js {     # 正则匹配
    root /webroot/res/; # 静态资源，root或alias
}
location / {     # 普通请求或/login等
    proxy_pass http://tomcat:8080/index # 动态请求
}
```

location 基本语法是：**location [=|~|~\*|^~|@] pattern{……}**。URL的配置规则：

- 以 = 开头，表示精确匹配；如只匹配根目录结尾的请求，后面不能带任何字符串。
- 以^~ 开头，表示uri以某个常规字符串开头，不是正则匹配。
- 以~ 开头，表示区分大小写的正则匹配。
- 以~* 开头，表示不区分大小写的正则匹配。
- !~和!~*分别是对上面两个的规则取非。
- @，定义命名location区段，这些区段客户端不能访问，只能内部访问。
- 以/ 开头，通用匹配, 如果没有其它匹配,任何请求都会匹配到。

查找顺序和优先级

1. 带有“=“的精确匹配优先 
2. 没有修饰符的精确匹配 
3. 正则表达式按照他们在配置文件中定义的顺序 
4. 带有“^~”修饰符的，开头匹配 
5. 带有“~” 或“~*” 修饰符的，如果正则表达式与URI匹配 
6. 没有修饰符的，如果指定字符串与URI开头匹配

实际使用的一般规则匹配的结果分两种

- 一种是静态资源，如js/css/img/html等，使用root、alias定向到具体的目录；root是最上层目录定义，location中的路径会累加；alias是别名定义，路径不累加直接查找定义的目录下文件。
- 另外一种是动态请求，代理到后端的服务。

关于代理的URL拼接部分：

- location后面的url加上了“/”，影响了匹配规则，对URL拼接来说，只是将整个URL切成了：已经匹配部分+其余部分。
- proxy_pass后面的url加上了“/”就相当于是绝对根路径。proxy_pass后有“/”，就不拼接location的已经匹配部分。如果它后面没有“/”，就全拼接，即已经匹配部分+其余部分。

## Rewrite

rewrite 是 nginx的静态重写模块，可在server,location,if中配置，而rewrite的模块中的return一旦执行就不会再处理接下来的模块了。

基本用法是 rewrite patten replace flag。patten是正则表达式，与patten匹配的URL会被改写为replace，flag可选。rewrite 后面可以加flag，flag标记有：

- **last** 相当于Apache里的[L]标记，表示完成rewrite，然后对重写的URI重新查找
- **break** 终止匹配, 不再匹配后面的规则，但是后续非rewrite语句可以执行
- **redirect** 返回302临时重定向 地址栏会显示跳转后的地址
- **permanent** 返回301永久重定向 地址栏会显示跳转后的地址

**if 指令**也是rewrite模块中的一个指令。其可作用于 server 和 location 块中，if指令不支持多条件、不支持嵌套、不支持else，语法是**if** (condition) { ... }。condition可以是如下类型：

- 变量名，如果变量的值是空字符串或者0表示false
- 可使用 字符串与变量做匹配 '=' 和 '!=' 。
- 也可以正则匹配 '~' 或 '~!'。
- 检测文件是否存在'-f' 或 '!-f' ，目录则是 '-d'。
- 检查文件,目录,软链接是否存在' -e' 或者 '!-e'。
- 检查是否为可执行文件，使用'-x' 或者 '!-x'。

**return指令**也是rewrite 模块提供的。基本语法是 return [code] [text]/URL;

**error_page核心指令**是当发生错误的时候能够显示一个预定义的uri。如error_page 404 /404.html; 或者error_page 404 =200 /empty.gif;（重置返回码）

## 变量

nginx的变量以符号$开始，变量分为两种，自定义变量与内置预定义变量，部分内置预定义的变量的值是可以改变的。自定义变量可以在sever,http,location等标签中使用set命令声明变量，语法如下“**set $变量名 变量值**”。所有的变量都是全局可见的（并不是全局变量），但是在不同的层级有各自的可见性。