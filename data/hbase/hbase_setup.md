# Hbase运维之安装与配置

Hbase的原型是Google的BigTable，2007年放出第一版v0.15.0，在Apache作为Hadoop的子项目来开发维护的。在2010 年发布v0.89.x 版本后 Hbase 成为 Apache 的顶级项目。2014年发布的v0.98.0版本是国内公司最早使用的版本，变化也比较大，相关文档最全面，数据结构在这个版本就基本完善了。

随后经历了四年的开发才在2018年发布的v2.0.0，这是一个非常重大的版本，多项新特性与Bug修复。比如内存数据块压缩；Netty的RPC server；RegionServer划分多个逻辑Group；小文件支持等。

其中0.98.0、1.2.6版本等版本应用比较广泛，稳定性、可靠性等都经受了很多考验。

## 依赖环境

HBase的运行环境也有三种模式：单机模式、伪分布模式和完全分布式模式。

- 单机模式，是在同一个JVM中运行所有Hbase守护进程和本地ZooKeeper。此模式下Hbase可以不使用HDFS，而是使用本地文件系统。
- 伪分布模式，所有守护进程都在单个节点上运行，都是独立进程。
- 完全分布式模式，所有进程都在不同的节点上独立运行，HBase的HMaster、HRegionServer等服务集群节点，和ZooKeeper集群与HDFS集群。

去官网下载安装包：hbase.apache.org/downloads.html。需要提前安装好ZooKeeper，与Hadoop，但是需要注意对应版本。

- hbase-2.2+已经可以很好的支持hadoop2.8+，hbase-2.4.x已经可以支持hadoop3.2.x。
- hbase-2.2+已经可以很好的支持JDK8，hbase-2.3+已经可以支持JDK11。
- ZooKeeper 3.4.x 可以支持HBase全部新特性。

## 安装与配置

安装配置Hbase前，需要先安装jdk、zookeeper，略。

下载Hbase安装包，编译好的安装包是.tar.gz文件，解压到指定目录，如/usr/local/。修改conf/hbase-env.sh文件，配置Java。

```shell
tar -xzf hbase-xxx-bin.tar.gz
cd hbase-xxx/
vi conf/hbase-env.sh
## 添加Java依赖
export JAVA_HOME=/usr/lib/jvm/java-xxx/
```

然后再根据不同的模式配置conf目录下的hbase-site.xml、regionservers、backup-masters、hadoop 的 hdfs-site.xml 和 core-site.xml。

最后配置环境变量，方便执行hbase命令。

```shell
vi /etc/profile
## 加执行命令目录
export HBASE_HOME=/usr/local/hbase-xxx
export PATH=$HBASE_HOME/bin:$PATH
```

启动。

```shell
## 启动命令：
bin/start-hbase.sh
## 关闭命令：
bin/stop-hbase.sh
```

分别来说几个模式的配置。

- 单机模式

使用自带zookeeper，使用本地文件系统存储数据。配置仅需要指定数据目录，配置hbase-site.xml。

```xml
<property>
    <name>hbase.rootdir</name>
    <value>file:///data/hbase</value>
</property>
<!--可选的zookeeper数据目录-->
<property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/data/zookeeper</value>
</property>
```

其中数据目录也可以是HDFS，如：hdfs://hadoop:9000/hbase。也可以不配置，默认当前目录下的./tmp中。

- 伪分布模式

还是使用自带zookeeper，使用HDFS做存储，但是配置方式与分布式相同。

hbase-env.sh中指定HBASE_MANAGES_ZK=true。

hbase-site.xml中指定副本数dfs.replication为1份。

regionservers中指定本机一个节点。

- 完全分布式模式

使用独立的zookeeper，使用HDFS做存储，多个节点。

配置hbase-env.sh

```shell
vim hbase-env.sh
## 使用自有zk
export HBASE_MANAGES_ZK=false
```

配置conf/hbase-site.xml，主要加hdfs与zk的地址

```xml
<property>
    <name>hbase.rootdir</name>
    <value>hdfs://hadoop:9000/hbase</value>
</property>       
<!--表示是一个分布式的环境-->
<property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
</property>   
<!--ZK的地址，加“,”可多个-->
<property>
    <name>hbase.zookeeper.quorum</name>
    <value>zkserver001</value>
</property>
<!--Region的冗余,只有一个RegionServer-->
<property>
    <name>dfs.replication</name>
    <value>2</value>
</property>
```

配置RegionServer。

```shell
vim conf/regionservers
## 加入本地节点
hbase001
hbase002
hbase003
```

需要注意的是必须保证多个节点时间一致。可以用ntpdate [ntpd服务器]。

启动master服务器，即可启动全部节点，当前机器为主节点，其他节点为从节点。需要打通ssh互通，略。

## 连接和访问

Hbase的Web UI，默认是：127.0.0.1:16010/master-status，可以Hbase的状态。

shell命令行访问，bin/hbase shell。常见的命令有：

```shell
help ## 帮助
create 'tableName', 'columnFamilyName'  ## 建表 表名和簇名
list 'tableName' ## 列出表信息
describe 'tableName' ## 查看表详情
put 'tableName', 'row1', 'columnFamilyName:a', 'value1' ## 新增数据，参数：表，行号，字段，值
scan 'tableName' ## 扫描所有表数据
get 'tableName', 'row1' ## 获取一行数据
drop 'tableName' ## 删除表
disable|enable 'tableName' ## 禁用/启用表
```

