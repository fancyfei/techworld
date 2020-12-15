# Hadoop常用文件及作用

Hadoop的完整的安装目录如下，常用的命令脚本放在bin与sbin下面，常用的配置放在etc/hadoop下面。

```
hadoop
├─bin
├─etc
│  └─hadoop
├─include
├─lib
│  └─native
├─libexec
├─sbin
└─share
```

bin目录放着Hadoop最核心的管理脚本和使用脚本。其他脚本的基础，也可以单独执行。

- hadoop，一切命令的核心，用于执行hadoop脚本命令，一般hadoop-daemon.sh调用执行。
- hdfs，用于执行hdfs脚本命令。
- yarn，用于执行yarn脚本命令。

sbin目录下的脚本则是调用bin目录下的脚本来实现的。

- hadoop-daemon.sh，通过执行hadoop命令来启动/停止一个守护进程(daemon)；该命令会被bin目录下面所有以start或stop开头的所有命令调用来执行命令；hadoop-daemons.sh也是通过调用hadoop-daemon.sh来执行命令；hadoop-daemon.sh本身就是通过调用hadoop命令来执行任务。
- start-dfs.sh，启动hdfs的NameNode、DataNode以及SecondaryNameNode。
- start-yarn.sh，启动Yarn的ResourceManager以及NodeManager。
- stop-dfs.sh，停止NameNode、DataNode以及SecondaryNameNode。
- stop-yarn.sh，停止ResourceManager以及NodeManager。
- start-all.sh，相当于执行 start-dfs.sh 及 start-yarn.sh。相应有stop-all.sh。

etc/hadoop目录，etc下只有一个目录，就是放各种配置的。

- hadoop-env.sh，设置Hadoop环境变量的脚本，主要是JDK，Hadoop配置项等等。
- core-site.xml，Hadoop全局配置文件，namenode路径、数据文件目录等等。
- hdfs-site.xml，HDFS的核心配置文件，副本数，数据存储目录等等。
- mapred-site.xml，MapReduce的核心配置文件，模板文件mapred-site.xml.template。
- slaves，用于设置所有的slave的名称或IP，每行存放一个。
- yarn-site.xml，yarn的核心配置文件，指定Yarn各组件路径等等。



