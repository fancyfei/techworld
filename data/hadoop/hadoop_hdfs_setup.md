# HDFS的配置与管理

HDFS（Hadoop Distributed File System）是Hadoop三个基础组件之一，为另外的组件以及大数据生态中的其他组件提供了最基本的存储功能，具有高容错、高可靠、可扩展、高吞吐率等特点。HDFS运行在java环境中，因此我们都需要安装JDK。安装完成之后是一个分布式网络文件系统，需要多节点协同组成Master/Slave模式。

## 安装

Hadoop版本的历史是2011年1.0+版，2012年2.0+可用，目前最新已经是3.0在2017年就发布了。安装包中包括了HDFS与Yarn组件，以及MapReduce计算框架，还有其他的基础工具包和 RPC 框架。

- 下载安装包，可以在官网https://hadoop.apache.org/releases.html，国内也有镜像。然后解压到一个目录。

- 安装JDK，新版的需要JDK1.8及以上。yum -y install jdk。

- JDK依赖配置，配置etc/hadoop/hadoop-env.sh文件的JAVA_HOME，默认是export JAVA_HOME=${JAVA_HOME}。/etc/profile中再加export JAVA_HOME=/usr/lib/jvm/java.xxx。

- Hadoop执行环境配置，将hadoop/bin和sbin加入到path中，/etc/profile中加：

  ```shell
  export HADOOP_HOME=/usr/local/hadoop-xxx
  export HADOOP_MAPRED_HOME=$HADOOP_HOME
  export HADOOP_COMMON_HOME=$HADOOP_HOME
  export HADOOP_HDFS_HOME=$HADOOP_HOME
  export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
  export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
  ```

## 配置

Hadoop部署方式分三种，Standalone mode（本地单独模式）、Pseudo-Distributed mode（伪分布模式）、Cluster mode（集群模式），其中前两种都是在单机部署，都是分别是开发与测试用途，生产环境使用集群模式。后面两种模式中包括的组件进程有：HDFS daemon的 NameNode（包括Sercondary NameNodes） 和 DataNode、YARN daemon的 ResourceManger 和 NodeManager，分别启动单独的java进程。

Hadoop解压并配置好环境之后，修改各配置文件。配置文件都是xml，所以都是这样子的：

```xml
<configuration>
    <property>
          <name>key</name>
          <value>value</value>   
    </property>
</configuration>
```

**core-site.xml** 全局参数，主要是NameNode来读取的配置，文件目录等。

- hadoop.tmp.dir，默认是/tmp/hadoop-${user.name}，生产需要修改。
- fs.defaultFS，文件系统的URI。通常是NameNode的hostname与port。如：hdfs://<your_namenode>:9000/ 。（单一NameNode，配置fs.default.name，有HA的配置fs.defaultFS）
- 日志：hadoop.logfile.size与hadoop.logfile.count，fs.trash.interval回收站清空时间，io.file.buffer.size读写缓存。

**hdfs-site.xml** HDFS的核心配置文件，副本数，数据存储目录等。

- dfs.name.dir，NameNode 元数据存放位置， 默认值：${hadoop.tmp.dir}/dfs/name。
- dfs.data.dir，DataNode 在本地磁盘存放block的位置，可多个，默认值：${hadoop.tmp.dir}/dfs/data。
- dfs.replication，DataNode上设置的副本份数，默认是3份，客户端也可以指定。
- dfs.blocksize，数据块大小，单位byte。默认是134217728（128M）。
- dfs.http.address，NameNode web管理地址与端口，默认9870。

**etc/hadoop/slaves** DataNode上读取的所有的slave的名称或IP，每行存放一个。

## 启动及异常处理

HDFS启动会独立NameNode、Sercondary NameNodes 和 DataNode 这三个进程。Secondary NameNode的整个目的是在HDFS中提供一个检查点，它只是NameNode的一个助手节点，定期唤醒，进行fsimage和edits的合并，防止文件过大。

- 生效环境配置变量：etc/hadoop/hadoop-env.sh
- 启动，sbin/start-dfs.sh
- 访问控制台，http://localhost:9870/，注意3.0版本的端口与之前不同。

遇到的常见异常及处理：

- 报用户问题Attempting to operate on hdfs namenode as root，hadoop-env.sh中加启动用户

  ```shell
  export HDFS_NAMENODE_USER="root"
  export HDFS_DATANODE_USER="root"
  export HDFS_SECONDARYNAMENODE_USER="root"
  export YARN_RESOURCEMANAGER_USER="root"
  export YARN_NODEMANAGER_USER="root"
  ```

- 大量的command not found，先要hdfs namenode -format，再直接 start-all.sh启动，不能加sh。

- 报localhost: Permission denied，ssh问题，生成并注册一下这个ssh key即可。

  ```shell
  ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
  cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
  ```

- localhost:9870/访问失败，关闭防火墙或配置hdfs-site.xml中的dfs.http.address为0.0.0.0:9870 。如有服务端口访问不到的，都可以检查一下IP是否是0.0.0.0，多网卡才会有的问题。

- 读写文件时遇到DataNode节点总是使用内网IP或localhost，解决办法有，一是使用域名，客户端指定属性"dfs.client.use.datanode.hostname", "true"，另外就是指定外网可访问IP启动节点服务。

- 调整参数后，DataNode启动不了，报clusterID不一致的错，把dfs/name/current/VERSION的clusterID复制到/dfs/data/current/VERSION中。

## 文件系统的使用

命令hadoop fs-help可以获取所有的基本的文件系统操作命令。如，hadoop fs -ls、-fsck等和hdfs dfs -ls、-mkdir、-cp等等。

访问http://localhost:9864/可以查看datanode节点上文件的占用情况等。

## 各组件端口

| 组件 | 节点     | 默认端口 | 配置                       | 用途说明                                                |
| ---- | -------- | -------- | -------------------------- | ------------------------------------------------------- |
| HDFS | DateNode | 9866     | dfs.datanode.address       | datanode服务端口，用于数据传输                          |
| HDFS | DateNode | 9864     | dfs.datanode.http.address  | http服务的端口                                          |
| HDFS | DateNode | 9865     | dfs.datanode.https.address | https服务的端口                                         |
| HDFS | DateNode | 9867     | dfs.datanode.ipc.address   | ipc服务的端口                                           |
| HDFS | NameNode | 9870     | dfs.namenode.http-address  | http服务的端口 ，管理端Web页面                          |
| HDFS | NameNode | 9871     | dfs.namenode.https-address | https服务的端口                                         |
| HDFS | NameNode | 9000     | fs.defaultFS               | 接收Client连接的RPC端口，用于获取文件系统metadata信息。 |







