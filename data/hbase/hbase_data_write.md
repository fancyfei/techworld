# Hbase作为实时系统的读过程

Hbase作为NoSQL数据库，不具备“强一致性”的事务处理能力，满足CAP理论中的CP（一致性与分区容错性），另外用了错误监控、主备模式以及Zookeeper的协调等技术来保障可用性。

Hbase的底层文件是放在HDFS上的，而HDFS文件设计的特点是追加模式，所以Hbase的使用场景也同样更适合在数据仓库类型业务。对于修改的数据是以更大的版本号（timestamp）追加到文件的。

## 写数据过程

写数据的过程，ZooKeeper -- meta -- regionserver-- Hlog -- MemStore -- storefile