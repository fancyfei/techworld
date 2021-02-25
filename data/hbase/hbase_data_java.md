# Hbase的程序操作

Hbase采用Java实现，其中 Java API 是原生支持的，其它编程语言接口需要通过 Thrift 协议支持。在HBase 官方代码包里含有Java实现的原生访问客户端，相关的类在 org.apache.hadoop.hbase.client 包中，都是与 Hbase 数据存储管理相关的 API。

## 相关依赖

官方的Jar包在HBase 安装目录下的 lib 子目录。但是一般还是使用Maven管理Jar依赖。主要用到hbase-client-x.x.x.jar和hbase-common-x.x.x.jar。

maven管理依赖，在项目的pom.xml加：

```xml
<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-client</artifactId>
    <version>2.4.1</version>
</dependency>
<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-common</artifactId>
    <version>2.4.1</version>
</dependency>
```

导入的基本类有：

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.*;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;
```

## API的类

管理 Hbase，则用 Admin 接口来创建、删除、更改表。数据的CRUD，则使用 Table 接口等。主要是Admin 和 Table 接 口以及 HBaseConfiguration、HTableDescriptor、HClounmDescriptor、Put、Get、Result、Scan 这些类。

- Admin、管理HBase数据库的表信息的接口。
- HBaseConfiguration、Hbase的配置类。
- HTableDescriptor、表的描述类，包含了表的名字及其对应表的列族。
- HClounmDescriptor、列族描述类，维护着关于列族的信息，例如版本号，压缩设置等。
- HTable、表的管理类，和HBase表直接通信。
- Put、数据新增修改管理类，对单个行执行添加操作。
- Get、数据获取管理类，对单个行执行查询操作。
- Result、结果类，存储Get或者Scan操作后获取表的单行值。
- Scan、限定查找类，对多个行执行查询操作。
- ResultScanner、获取scan值的接口类。

## Java客户端程序

- 建立连接，在分布式环境下，客户端访问 HBase 需要通过 ZooKeeper 的地址和端口来获取当前活跃的 Master 和所需的 RegionServer 地址。在Hbase3.0+中，可以直接使用Master地址。

```java
Configuration cf = HBaseConfiguration.create();
//默认也是2181
// configuration.set("hbase.zookeeper.property.clientPort", "2181");
cf.set("hbase.zookeeper.quorum", "10.0.3.71");
// v3.0可只配置master地址。
// cf.set("hbase.master", "10.0.3.71:16000");
Connection conn = ConnectionFactory.createConnection(cf);
//使用完后，需要关闭连接
void close();
```

- 表操作，都是使用 Admin 接口，必须调用 Connection.getAdmin() 方法返回一个子对象。

```java
String tablename = "TableName";
TableName tableName = TableName.valueOf(tablename);
Admin admin = conn.getAdmin();
//Admin常用的接口如下：
//表是否存在
boolean tableExists(TableName tableName);
//列出所有表
List<TableDescriptor> listTableDescriptors();
//删除表，先禁用表，再删除
void disableTable(TableName tableName);
void deleteTable(TableName tableName);
//启用已禁用的表
void enableTable(TableName tableName);
//给表加字段
void addColumnFamily(TableName tableName, ColumnFamilyDescriptor columnFamily);
//创建表，需要指定列族
String columnname = "cf";
List<ColumnFamilyDescriptor> families = new ArrayList<>();
ColumnFamilyDescriptor cfDesc = ColumnFamilyDescriptorBuilder.newBuilder(columnname.getBytes()).build();
families.add(cfDesc);
TableDescriptor tabDesc = TableDescriptorBuilder.newBuilder(tableName).setColumnFamilies(families).build();
admin.createTable(tabDesc);
admin.getConnection().close();
```

- 数据访问，通过Table对象连接到Hbase进行操作。

  新增数据如下：

```java
//需要先有Table与TableName实例
Table table = conn.getTable(tableName);
//一行，参数是rowKey
Put put = new Put ("rowKeyName".getBytes());
//加字段值：参数分别是列族，列，值
put.add("cfName".getBytes(),"colName".getBytes(),"value001".getBytes());
table.put(put);
```

获取单行数据如下：

```java
Get get = new Get("rowKeyName".getBytes());
Result result = table.get(get);
//从结果中拿到单元值
for (Cell cell : result.rawCells()) {
    //列族名 cell.getFamilyArray()
    //列名 cell.getQualifierArray()
    //值 cell.getValueArray()
    s
}
```

限定查找，可指定版本号、起始行号、终止行号、列族、列限定符、返回值的数量的上限等：

```java
Scan scan = new Scan();
//可以指定scan参数：addColumn、setTimeRange、setBatch
ResultScanner resutScanner = table.getScanner(scan);
//拿到结果集
for(Result result: resutScanner){
    //对结果进行列值提取 result.rawCells()
}
```

## 客户端连接时可能的报错

  - NoNode for /hbase/master
    master没有正常启动，将hbase的master节点重启一下。
  - Master is initializing
    master在zookeeper中的数据不正常，删除后，重启一下hbase。hdfs中的文件也要删除一下。