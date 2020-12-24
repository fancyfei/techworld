# HDFS的数据存储结构

HDFS有三个独立的服务，NameNode、Sercondary NameNodes 和 DataNode 。一般在生产环境是分别在不同的服务器上启动各自服务。启动方式如hdfs --daemon start/stop  datanode/namenode/journalnode（journalnode是NameNode的备份），启动namenode时自动启动NameNode和Sercondary NameNodes进程。

Secondary NameNode的整个目的是在HDFS中提供一个检查点，它只是NameNode的一个助手节点，定期唤醒，进行fsimage和edits的合并，防止文件过大。

下面展示了三个节点的数据目录结构。

```shell
hadoop-root
└── dfs
    ├── data
    │   └── current
    │       └── BP-666131917-127.0.0.1-1608040646449
    │           ├── current
    │           │   ├── finalized
    │           │   └── rbw
    │           └── tmp
    ├── name
    │   └── current
    └── namesecondary
        └── current
```

## NameNode的数据目录

```shell
name
├── current
│   ├── edits_0000000000000000001-0000000000000000002
│   ├── edits_......
│   ├── edits_inprogress_0000000000000001209
│   ├── fsimage_0000000000000000000
│   ├── fsimage_0000000000000000000.md5
│   ├── seen_txid
│   └── VERSION
└── in_use.lock
```

- Checkpoint概念，永久性检查点，用来定期将edits文件合并到新的fsimage中。
- 日志数据块，edits_start transaction ID-end transaction ID，编辑日志文件，一次事务一个（上一个Checkpoint到下一个Checkpoint），记录事务中所有操作。
- edits_inprogress，edits_inprogress__start transaction ID，当前正在被追加的edit log。同一时刻只有一个打开可写的编辑日志。重启NameNode加载fsimage后，会读此文件中的日志，用来保持和关闭前数据一致。
- fsimage块，fsimage_end transaction ID，文件系统镜像文件，所有的目录和文件元数据。重启NameNode首先从这儿加载数据到内存。同名.md5的文件是作完整性校验的。
- VERSION，版本号。
- seen_txid，保存最近一次fsimage或者edits_inprogress的transaction ID。
- in_use.lock，进程锁，防止多个进程同时启用的冲突。

## DataNode的数据目录

```shell
data
├── current
│   ├── BP-666131917-127.0.0.1-1608040646449
│   │   ├── current
│   │   │   ├── dfsUsed
│   │   │   ├── finalized
│   │   │   │   └── subdir0
│   │   │   │       └── subdir0
│   │   │   │           ├── blk_1073741857
│   │   │   │           ├── blk_1073741857_1033.meta
│   │   │   │           ├── blk_...... ## 下一个文件块
│   │   │   ├── rbw
│   │   │   └── VERSION
│   │   ├── scanner.cursor
│   │   └── tmp
│   └── VERSION
└── in_use.lock
```

- 目录名是BP-random integer-NameNode-IP address-creation time。用了VERSION中的集群唯一blockpoolID。
- finalized中的数据块，用于实际存储HDFS BLOCK的原始字节数据，每个实际文件对应一个块文件。对应的.meta文件包含了头部和checksum信息。数据块的数量到达一定的规模的时候会新建一个目录。
- rbw，用于存储用户当前正在写入的数据。
- VERSION，版本号。
- in_use.lock，进程锁，防止多个进程同时启用的冲突。

## SecondaryNameNode的数据目录

```
namesecondary
├── current
│   ├── edits_0000000000000000001-0000000000000000198
│   ├── edits_......
│   ├── fsimage_0000000000000000397
│   ├── fsimage_0000000000000000397.md5
│   ├── fsimage_......
│   └── VERSION
└── in_use.lock
```

Secondary NameNode进行合并操作的步骤是，首先将NameNode的fsimage和edits下载到自已的数据目录，然后合并到fsimage文件，最后上传到NameNode节点。所以它的目录与NameNode类似。

## 各节点的VERSION

```properties
## name & namesecondary
namespaceID=1889662825
blockpoolID=BP-666131917-127.0.0.1-1608040646449
storageType=NAME_NODE
layoutVersion=-65
## data
datanodeUuid=3ef22e6c-01f8-4009-89c7-37050190698b
storageID=DS-b2121a95-1ad8-49dc-81e4-12a7c6c41616
storageType=DATA_NODE
layoutVersion=-57
## 都有的字段，且clusterID要一致。
cTime=0
clusterID=CID-d49189d4-50e9-42b1-b04b-94adca89b6d7
```

- clusterID，整个集群唯一的标识。format namenode会生成一个新的。但是可以手工指定。
- namespaceID，文件系统命名空间的唯一标识，集群中全局唯一，一个namenode管理一个命名空间。
- blockpoolID，是数据块池的唯一标识符，集群中全局唯一，数据块池中包含了由一个namenode管理的命名空间中的所有文件。
- datanodeUuid，也是集群中全局唯一，各数据节点的唯一标识。
- storageType - 有两种取值NAME_NODE /DATA_NODE /JOURNAL_NODE。
- cTime，HDFS创建时间，在升级后会更新该值。
- layoutVersion，HDFS metadata版本号。


