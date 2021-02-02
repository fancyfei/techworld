# Hbase作为实时系统的读过程

HBase不是一个关系型数据库，不具备“强一致性”的事务处理能力，也不能处理固定模式与复杂数据关系。它是运行在Hadoop上的NoSQL数据库，是一个分布式的和可扩展的大数据仓库，因此适合的场景是松散数据，并支持高并发访问，如果数据量不大，读取数据性能。

HBase支持Hadoop map-reduce程序设计模型。HBase有两种访问方式：通过行键进行随机访问；通过map-reduce脱机或批访问；前者主要提供了get、scan以及filter等方法读取数据库中的数据。

## Hbase数据结构与架构

HBase 的数据模型的核心是表，包含着行、键列族和列。get与scan都以行键rowkey 为参数查找数据位置并读取。为了减少读取的IO次数，加了多层缓存，首先表的元数据将缓存在客户端，HRegion上也先访问内存中的memstore以及blockcache。另外Hfile为行键设置了多种索引。

- Hbase Table数据结构组成：Table = rowkey + family + column + timestamp + value。

- Hbase的结构元素组成：ZooKeeper，HMaster，HRegionSever，HRegion，HStore，MemStore与StoreFile，HFile。

## 读数据过程

Hbase的读操作参与角色有：ZooKeeper -> meta -> HRegionServer -> HRegion -> memstore -> Storefile。

HBase一次范围查询可能会涉及多个Region、多块缓存甚至多个Hfile。HBase中更新操作使用时间戳属性实现了多版本，删除操作也只是插入了一条标记为"deleted"标签的数据，读取过程需要过滤版本、标记删除的数据。

总体的读流程如下：

1. Client访问zookeeper，获取元数据hbase:meta存储所在的HRegionServer地址。
2. 根据地址访问对应的HRegionServer，拿到hbase:meta加载到内存中，用RowKey找到表存储的一个或多个HRegionServer地址。
3. 去表所在的HRegionServer，发起数据的读取请求。
4. server查找对应的HRegion，在HRegion中寻找列族，先找到memstore，找不到去blockcache中寻找，再找不到就会访问磁盘中HFile读取数据。
5. 找到数据之后会先缓存到blockcache中，再将结果返回给Client。（blockcache逐渐满了之后，会采用LRU的淘汰策略。）

当Client的数据读取请求到HRegionServer后，Scanner在server上的流程分解如下：

![hbase_data_read_class_process](hbase_data_read_class_process.png)

1. HRegionServer构建RegionScanner，用于对该Region的数据检索。
2. 一个RegionScanner管理一堆ColumnFamily，构造StoreScanner，用于对该列族的数据检索。多个StoreScanner合并构建最小堆（已排序的完全二叉树）StoreHeap:PriorityQueue<StoreScanner>。
3. StoreScanner管理一堆HFile，构造一个MemStoreScanner和一个或多个StoreFileScanner（数量取决于StoreFile数量），这个是实际读取数据的地方（除了compaction状态的HFile）。需要过滤掉RowKey一定不在StoreFile内的对应的Scanner，主要过滤策略有：Time Range过滤、Rowkey Range过滤以及布隆过滤器。
4. 每个Scanner seek到startKey，这个步骤就是在每个StoreFile或MemStore中seek扫描起始点startKey。
5. 将所有的StoreFileScanner和MemStoreScanner合并并由小到大排序，构建最小堆KeyValueHeap:PriorityQueue<KeyValueScanner>。
6. 通过触发Scanner的next() 调用，来遍历所有 Scanner 中的数据，将结果返回。

scanner.next的步骤分解如下：

1. Scan是一行一行获取的，获取完一行之后再获取下一行。
2. 一般查找的顺序是先memstore，然后blockcache，最后到HFile。
3. 每次拿一次数据都要判断下是否有stopRow，如果有可以停在搜索返回。
4. 最终返回的List<Cell>results要封装成Result[]格式返回。



## 附：数据部分的KeyValue数据结构

KeyValue由Key length、value length、key、value组成。其中key又由RowKey length、RowKey、ColumnFamily length、ColumnFamily、ColumnQualifier、TimeStamp、KeyType组成。

- RowKey length：RowKey长度
- RowKey：RowKey内容
- ColumnFamily length：列族长度
- ColumnFamily：列族
- ColumnQualifier：列名
- Timestamp：时间戳
- KeyType：表示类型，取值有Put/Delete/Delete Column/Delete Family



## 附：rowkey设计

rowkey的设计上需要满足长度原则、散列原则、唯一原则。

长度原则：建议是越短越好，不要超过16个字节。

散列原则：建议将Rowkey的高位作为散列字段，减少RegionServer上堆积的热点现象。

唯一原则：必须在设计上保证其唯一性，因为rowkey是按照字典顺序排序存储的。

