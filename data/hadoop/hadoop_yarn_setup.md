# Yarn 的安装与配置

Yarn是Hadoop的三大核心组件之一，从hadoop 2引入，来优化MapReduce程序调度的。在Hadoop程序包中，就包含有Yarn的NodeManager、ResourceManager两个子组件，可以分别在不同的机器上单独启动进程。

## 安装与配置

下载Hadoop包并解压，就完成了安装，也可以选择重新编译源码来得到安装包。

- 安装JDK8+，配置JDK的环境变量等等。

- 配置Hadoop的环境：配置etc/hadoop/hadoop-env.sh文件的JAVA_HOME和；

  ```shell
  export HADOOP_HOME=/usr/local/hadoop-xxx
  export HADOOP_COMMON_HOME=$HADOOP_HOME
  export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
  export YARN_HOME=$HADOOP_HOME
  export HADOOP_MAPRED_HOME=$HADOOP_HOME
  export YARN_CONF_DIR=$HADOOP_HOME/etc/Hadoop
  ```

  将hadoop/bin和sbin加入到path中export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH，这可以放在/etc/profile中，也可以在hadoop-env.sh。

- 配置各个机器的SSH免密登陆。

Hadoop配置文件都是xml，都在etc/hadoop/目录下，所有XML内容都是下面的格式：

```xml
<configuration>
    <property>
          <name>key</name>
          <value>value</value>   
    </property>
</configuration>
```

**yarn-site.xml**，Yarn的站点配置文件。主要是配置的property参数是：

- yarn.resourcemanager.hostname，RM的主机名。
- yarn.nodemanager.aux-services，指定NM的附属服务，需配置成mapreduce_shuffle，才能运行MR程序。
- yarn.resourcemanager.cluster-id，唯一的集群的名称。集群部署需要指定。
- yarn.application.classpath，Hadoop预置JAR包的目录。通过命令hadoop classpath是需要的classpath。

下来配置一般情况下默认即可。

- yarn.resourcemanager.address、yarn.nodemanager.address，节点对客户端暴露的地址，前者默认8032。
- yarn.resourcemanager.scheduler.address，RM申请资源、释放资源的地址。
- yarn.resourcemanager.webapp.address，用户访问的web管理地址。
- yarn.resourcemanager.scheduler.class，启用的资源调度器主类。目前可用的有默认的Capacity Scheduler以及FIFO和Fair Scheduler。
- yarn.resourcemanager.bind-host与yarn.nodemanager.bind-host，节点的IP绑定。

**mapred-site.xml**，MapReduce的配置文件。

- mapreduce.framework.name，指定使用yarn运行mapreduce程序。

如果需要有Job历史服务器就配置下面参数

- mapreduce.jobhistory.address，JobHistory服务器IPC地址
- mapreduce.jobhistory.done-dir，Hadoop作业历史记录保存位置

## 启动

启动|停止单个节点的NodeManager、ResourceManager。以及jobhistoryserver。

```shell
## 启动或关闭单个服务
yarn --daemon stop|start nodemanager/ResourceManager
## 一把全部启动或关闭
start-yarn.sh|stop-yarn.sh
## 需要启动Job的历史服务器时，单独启动historyserver
mr-jobhistory-daemon.sh start historyserver
```

注：Hadoop通过jobhistoryserver来查看作业和历史。

访问：http://master_ip:8088/ 登陆到Yarn的后台管理端，可以查看任务的情况、节点的状态等。

## 默认值与端口

| 组件 |       节点        | 默认值/端口 | 配置                                          | 用途说明                                |
| :--: | :---------------: | :---------: | --------------------------------------------- | --------------------------------------- |
| Yarn |  resourcemanager  |   0.0.0.0   | yarn.resourcemanager.hostname                 | RM的hostname                            |
| Yarn |  resourcemanager  |    8032     | yarn.resourcemanager.address                  | RM的地址，提交、杀死程序                |
| Yarn |  resourcemanager  |    8030     | yarn.resourcemanager.scheduler.address        | 调度器地址，AM申请、释放资源            |
| Yarn |  resourcemanager  |    8031     | yarn.resourcemanager.resource-tracker.address | NM向RM领取任务，心跳                    |
| Yarn |  resourcemanager  |    8088     | yarn.resourcemanager.webapp.address           | YARN的WEB U地址，对应https是8090        |
| Yarn |  resourcemanager  |             | yarn.resourcemanager.scheduler.class          | 启用的资源调度器主类                    |
| Yarn |  resourcemanager  |    8033     | yarn.resourcemanager.admin.address            | RM管理员地址                            |
| Yarn |    nodemanager    |   0.0.0.0   | yarn.nodemanager.hostname                     | DM的hostname                            |
| Yarn |    nodemanager    |    8041     | yarn.nodemanager.address                      | NM中container manager的端口             |
| Yarn |    nodemanager    |    8042     | yarn.nodemanager.webapp.address               | DM的WEB UI端口，对应https的是8044       |
| Yarn |    nodemanager    |             | yarn.nodemanager.aux-services                 | NM的附属服务，需配置成mapreduce_shuffle |
| Yarn | JobHistory Server |    10020    | mapreduce.jobhistory.address                  | JobHistory服务器IPC地址                 |
| Yarn | JobHistory Server |    19888    | mapreduce.jobhistory.webapp.address           | JobHistory服务器Web UI地址              |


