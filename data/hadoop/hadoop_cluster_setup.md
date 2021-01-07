# 搭建hadoop分布式集群环境

一般情况下Hadoop集群的组成主要有HDFS集群和Yarn集群。

一个HDFS集群由一个NameNode节点和多个DataNode节点组成，至少需要3个节点组成一个健康的HDFS集群。另外还有两个节点是：助手节点Secondary NameNode和HA模式下NameNode的备份JournalNode，SNN可以独立也可以与NN放一起，高可用的JournalNode就需要独立启动才有意义（至少3个节点或以上奇数个）。

一个Yarn集群由一个ResourceManager节点和多个NodeManager节点组成（Master/Slave结构），同样一般也是3个节点组成。Yarn集群也有一 个节点：历史记录服务器JobhistoryServer，默认关闭，也可独立启动，或与RM放在一起。ResourceManager的HA用到的是zookeeper服务来实现的。

Yarn与HDFS属于两个不同的集群，一个负责文件存储，一个负责作业调度，但是通常会把NodeManager和DataNode放在一起是尽可能使计算作业就在存储节点上执行（“计算向数据移动”）。

## 两个集群节点清单

|      | 节点                            | HA模式下增加的节点 |
| :--: | :------------------------------ | :----------------- |
| HDFS | NameNode                        | JournalNode        |
| HDFS | DataNode1，DataNode2，...       |                    |
| HDFS | Secondary NameNode              |                    |
| Yarn | ResourceManager                 | ZooKeeper集群      |
| Yarn | NodeManager1，NodeManager2，... |                    |
| Yarn | JobhistoryServer                |                    |

*ZooKeeper是Hadoop生态中解决节点高可用的另外的技术，集群部署的话也同样需要至少3个节点或以上的奇数个，略。*

## 环境需求

- 主机，linux版本如centos7.3，至少要能组3个节点，将Yarn与HDFS启动在同节点上，建一个专用用户，全程用此用户操作。创建数据目录。

- JAVA，至少JDK1.8+，配置JAVA_HOME 环境变量。

  ```sh
  sudo yum install java-11-openjdk.x86_64 java-11-openjdk-devel.x86_64
  ## 加入JAVA_HOME，也可是用户文件 .bashrc
  vi /etc/profile 
  ## 文件最后增加：
  export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.9.11-0.el7_9.x86_64
  export PATH=$PATH:$JAVA_HOME/bin
  ## 生效：或source ~/.bashrc
  source /etc/profile
  ## 创建目录
  mkdir -p /data/hdfs/data; mkdir -p /data/hdfs/name; mkdir -p /tmp/hadoop; 
  ```

- 网络，同一局域网，固定IP，和域名解析（修改hosts文件或有DNS服务器）hadoop依赖域名找节点的。包括客户端也要DNS解析。

  ```shell
  ## 固定IP设置在/etc/sysconfig/network-scripts/ifcfg-... 的文件中
  ## 主机名在/etc/hostname，快速生效下面命令，在不同机器上操作名称不同。
  hostnamectl set-hostname hadoop201
  ## 域名用hosts举例，各节点一致，包括客户端的主机。
  vi /etc/hosts
  ## 主机名与ip映射
  192.168.16.201 hadoop201
  192.168.16.202 hadoop202
  192.168.16.203 hadoop203
  ```

- 集群启动时，用到了SSH，需要配置当前用户ssh免密登陆到各个节点机器，包括自己。

  ```sh
  ## 生成公钥
  ssh-keygen -t rsa
  ##授权公钥,最后把所有节点上的authorized_keys内容累加一下
  cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
  ```

- 安装Hadoop，可去hadoop.apache.org下载编译好的包，直接解压，然后配置环境变量。

  ```shell
  tar -zxf hadoop-3.3.0.tar.gz -C /usr/local
  ## 加入HADOOP_HOME
  vi /etc/profile ## 文件结尾加入
  export HADOOP_HOME=/usr/local/hadoop-3.3.0
  export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
  ## 生效
  source /etc/profile
  ```
  
- Hadoop的jvm参数，配置在hadoop-env.sh中，略。各组件环境变量配置，以及SSH非22端口，还需配置这个HADOOP_SSH_OPTS。

  ```shell
  export HADOOP_HOME=/usr/local/hadoop-3.3.0
  export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
  export HADOOP_HDFS_HOME=$HADOOP_HOME
  export YARN_HOME=$HADOOP_HOME
  export HADOOP_MAPRED_HOME=$HADOOP_HOME
  export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
  ## 指定启动用户
  export HDFS_NAMENODE_USER=root
  export HDFS_DATANODE_USER=root
  export HDFS_SECONDARYNAMENODE_USER=root
  export YARN_NODEMANAGER_USER=root
  export YARN_RESOURCEMANAGER_USER=root
  ## ssh参数
  export HADOOP_SSH_OPTS="-p 62233"
  ```
  
  

## 各服务的配置文件

各配置文件结构类似。在各自的xml文件中配置<configuration>中的<property>。各个节点配置相同，可以scp复制到各节点。

- core-site.xml，配置全局参数，主要是hdfs集群名，就是NameNode的RPC地址，和临时目录。HA配置的zookeeper也在这儿配置，略。

  ```xml
  <property>
      <name>fs.defaultFS</name>
      <value>hdfs://hadoop201:9000</value>
  </property>
  <property>
      <name>hadoop.tmp.dir</name>
      <value>/tmp/hadoop</value>
  </property>
  <!--用于HA的ha.zookeeper.quorum 略 -->
  ```
  
- 日志文件大小可在core-site.xml配置，也可默认就行。存放目录默认在$HADOOP_HOME/logs，需要在log4j.properties配置hadoop.log.dir。log4j配置略。

- hdfs-site.xml，hdfs节点配置：备份数、数据目录，web管理端口（默认也行）。还有NN的HA配置，略。

  ```xml
  <property>
      <name>dfs.namenode.http-address</name>
      <value>hadoop201:9870</value>
  </property>
  <property>
      <name>dfs.replication</name>
      <value>2</value>
  </property>
  <property>
      <name>dfs.datanode.data.dir</name>
      <value>/data/hdfs/data</value>
  </property>
  <property>
      <name>dfs.namenode.name.dir</name>
      <value>/data/hdfs/name</value>
  </property>
  <!--NN的HA配置 略 -->
  ```
  
- mapred-site.xml，mapreduce程序的配置、历史服务器配置，需要指定Yarn做调度。mapreduce.application.classpath默认找$HADOOP_MAPRED_HOME环境变量配置的目录，如果没有配置就需要在此文件中指定全路径。

  ```xml
  <property>
      <name>mapreduce.framework.name</name>
      <value>yarn</value>
  </property>
  <!--mapreduce.application.classpath与jobhistory 略 -->
  ```

- yarn-site.xml，Yarn集群的配置，指定RM的位置，MR程序获取数据的方式等。还有RM的HA配置，多个RM的Host及zookeeper集群，略。

  ```xml
  <property>
      <name>yarn.resourcemanager.hostname</name>
      <value>hadoop201</value>
  </property>
  <property>
      <name>yarn.nodemanager.aux-services</name>
      <value>mapreduce_shuffle</value>
  </property>
  <!--RM的HA配置及zk-address 略 -->
  ```

- works，hdfs的DataNode节点清单，也是yarn的NodeManager节点清单。

  ```shell
  hadoop201
  hadoop202
  hadoop203
  ```

## 集群启动

主节点上启动HDFS的NameNode、Secondary NameNode、DataNode，以及Yarn的ResourceManager、NodeManager，从节点上启动HDFS的DataNode和Yarn的NodeManager。

- 在主节点启动，Hadoop会通过ssh将整个集群启动：

  ```shell
  ## NN节点需要格式化
  hdfs namenode -format
  start-dfs.sh
  start-yarn.sh
  ## 停止对应的是stop....sh
  ## 修改重要NN、RM等节点名称后，需要清理之前的数据，或者修改集群ID
  rm -rf /data/hdfs/data/*; rm -rf /data/hdfs/name/*; rm -rf /usr/local/hadoop-3.3.0/logs/*
  ```

- 如果需要启动单个服务：

  ```shell
  ## hdfs: 单独起NN、DN、2NN
  bin/hdfs --daemon start namenode | datanode | secondarynamenode
  ## yarn: 单独起RM、NM
yarn --daemon start resourcemanager | nodemanager
  ## HistoryServer: 单独启动历史服务器
  mapred --daemon start historyserver    
  ## 对应的停止是... stop ...
  ```
  
  





