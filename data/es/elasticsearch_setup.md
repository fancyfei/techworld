# Elasticsearch 部署与配置

Elasticsearch（ES） 是一个分布式、可扩展、实时的搜索与数据分析引擎。通过集群部署可以提高系统的可用性。数据被划分到不同的分片（shard）中，然后将shard分配到不同的节点上，实现分布式存储。

每启动一个ES服务就是一个实例，一台机器可以启动多个实例，也可以多个实例协同起来为一组，单个 ES 实例称为一个节点（node），一组节点构成一个集群（cluster）。一个节点启动后，就会使用Zen Discovery机制去寻找和连接上集群中的其他节点，集群中会从候选主节点中选举出一个主节点。

## 节点类型

用于生产环境的ES服务，部署的是多实例的集群模式，部署时需要配置主节点与数据节点，以及协调和Ingest角色等，不同类型的节点对资源的需求不同，根据不同的业务场景，选择不同的节点部署架构。

- 候选主节点（Master-eligible node），节点参数设置node.mater为true的都成为候选主节点，并参与选举出一个主节点（Master node）。主节点负责创建索引、删除索引、分配分片、追踪集群中的节点状态等工作。当我们设置node.data为false，就将节点设置为专用的候选主节点了。主节点是轻量型，计算量与IO消耗最少。
- 数据节点（Data node），节点参数设置node.data为false的都成为数据节点。负责数据的存储和相关具体操作，比如CRUD、搜索、聚合。需要足够的存储空间，如果读取较多的业务场景需要CPU、内存IO性能较好的资源。
- 客户端节点（Client node），既不做候选主节点也不做数据节点的客户端节点，只负责请求的分发、汇总等功能。独立的客户端节点对并发要求较高。
- 协调节点（Coordinating node），是个角色，所有节点都是协调节点。充当协调角色的节点对CPU、Memory和I/O要求比较高。
- Ingest node，是个角色，拦截批量和索引请求，做数据前置处理转换的节点。任何节点都可以处理Ingest任务，也可创建专用Ingest节点。设置为node.ingest：false/true。

## 环境准备

Elasticsearch对硬件的要求主要在内存与硬盘，对CPU需求不大。在分布式系统中，尽量在同一数据中心，集群如果跨数据中心则需要快速可靠的网络支撑。

- 内存：排序和聚合都是在内存中进行，对此类业务较多时，对内存有很大消耗，写入时需要的系统系统高速缓存也是提高性能的关键。
- 硬盘：一般来说，主要是存储的容量需求较大，并且防止单点故障需要多个副本，同时也能提高效率。但是数据量较大情况下，大并发量的写入会让硬盘成为集群的性能瓶颈，这时就推荐使用SSD。

Elasticsearch的官网是www.elastic.co，有较完善的英文文档。去官方下载安装包如elasticsearch-x.x.x.tar.gz。

安装前需要规划好资源：

- 安装目录如：/usr/local；和数据目录如：/data/elasticsearch/，其下再分/data与/logs。
- 独立的用户如：es，因为不让用root启动ES服务。从安装开始就使用此用户进行操作，会避免很多权限错误。
- 集群机器：可以让data和master混合部署，如果需要较多的data节点那么可以分离部署。
- ES将占用两个端口，9200作http外部通讯，9300做tcp内部通讯。

## 部署配置

安装过程，在centos7上举例，ES7+已经内置了OpenJDK，不需要再安装：

```shell
## 新增用户
useradd es
passwrd es
####输入两次密码
## 文件夹授权
chmod -R o+r+w /usr/local
chmod -R o+r+w /data
## 用新用户安装
su es
mkdir -p /data/elasticsearch/data
mkdir -p /data/elasticsearch/logs
tar -zxvf elasticsearch-7.11.1-linux-x86_64.tar.gz
```

配置过程，主配置文件是config/elasticsearch.yml。

```properties
## 集群配置：同一集群名组成同一个集群
cluster.name: es-cluster-01
discovery.seed_hosts: ["x.x.x.1", "ip2", "ip3"]
cluster.initial_master_nodes: ["es-node-01"]
## 本机配置：每台机器名不同
## 在同一集群中唯一的节点名
node.name: es-node-01
## # 是否主节点
node.master: true
## # 是否数据节点
node.data: true
path.data: /data/elasticsearch/data
path.logs: /data/elasticsearch/logs
network.host: 0.0.0.0
http.port: 9200
http.cors.enabled: true
http.cors.allow-origin: "*"
## 权限组件xpack配置，略
```

在config/jvm.options配置JVM内存参数。或者在config/jvm.options.d/目录下.options文件。

```properties
-Xms2g
-Xmx2g
```

操作的系统参数调优：

用户最大可创建文件数，vi /etc/security/limits.conf。

```shell
* soft nofile 65536
* hard nofile 131072
```

最大虚拟内存，vi /etc/sysctl.conf，保存后生效需要sysctl -p 。

```shell
vm.max_map_count=655360
```

## 服务的运维

直接脚本启动：

```sh
## 用专用用户启动
su es
## 启动，-d后台启动
./bin/elasticsearch -d
## 停止
kill -9 xxxx
```

注册为service服务，/etc/systemd/system目录下创建elasticsearch.service文件：

```sh
[Unit]
Description=elasticsearch
[Service]
User=es
LimitNOFILE=100000
LimitNPROC=100000
ExecStart=/usr/local/elasticsearch-7.11.1/bin/elasticsearch
[Install]
WantedBy=multi-user.target
```

注册为服务后的启动与停止：

```shell
## service的开机自启
systemctl enable elasticsearch
## 启动
service elasticsearch start
## 停止
service elasticsearch stop
```

注意集群节点的重启要先暂停shard自动均衡，服务节点重启之后再启用。allocation参数为none停止，all则是启用。

```shell
curl -XPUT http://x.x.x.x:9200/_cluster/settings -d'{
    "transient" : {"cluster.routing.allocation.enable" : "none"}
}'
```

增加节点到集群，只需要保持新节点的cluster.name与集群一致，会自动加入到集群中。但是如果是候选主节点，则需要把集群中所有的discovery.seed_hosts，需要逐个重启一遍。

访问验证：

web： x.x.x.1:9200/

命令： curl -XGET 'x.x.x.1:9200/?pretty'