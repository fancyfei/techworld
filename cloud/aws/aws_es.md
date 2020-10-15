# Amazon ES托管服务

Amazon 提供的Elasticsearch 集群，可自动检测和替换失败的节点，可大量用在搜索、日志分析、监控数据等，并且还提供了SQL支持（Open Distro）。但是NoSQL的功能一般还是建议使用DynamoDB 这个完全托管的 NoSQL 数据库服务。AmazonES有较低成本的存储，多级别的安全选择，并能做到多区域可用。另外，还默认安装了 Kibana。

## Elasticsearch的基本概念

- 集群

  cluster（集群），一个ES集群是由多个节点(Node)组成的，每个集群都有一个cluster name 作为标识。

- 节点

  node（节点），一个ES实例就是一个node，一个机器也可以有多个实例，最佳是部署一个。节点可以动态增加。

- 索引

  index（索引），即一系列文档（documents，数据）的集合。许多条 Document 构成了一个 Index。

- 文档

  document（文档），Index 里面单条的记录称为 Document（文档），同一个index下的Document，不要求有相同的结构（scheme），存储的格式为JSON。ES主要是对document中的field（字段）进行处理，如分词、倒排索引等。

- 分片

  shard（分片），每个索引有一个或多个分片，索引的数据被分配到各个分片上，主要目标是横向扩展与负载均衡。

- 副本

  replica（副本）可以理解为备份分片，主要职责是容错与负载读请求。ES会自动平衡副本在node中的位置。

## AWS的ES 域

Amazon ES 域指的就是cluster（集群），由数据节点（一个或多个可用区的实例）与存储资源等组成。当修改域配置时，会使用蓝绿部署方式进行。

- 数据节点

  数据节点中指定了：计算、内存（实例类型来确定的），和存储（如EBS）。node（实例）会在一个或多个可用区内创建，推荐多个可用区。如果启用了多可用区，则集群中的索引至少创建一个副本。t2.small+10G存储是免费的。

- 专用主节点

  专用主节点执行群集管理任务，但不保留数据也不响应数据上传请求。

- UltraWarm 数据节点

  UltraWarm（暖存储）也是数据节点，以经济高效的方式存储大量只读数据，使用方案是S3+缓存，所以也没有副本。

- 安全

  数据安全是使用多可用区，快照等来保障。网络访问安全由VPC与子网与安全组来控制，可以指定IP或IAM 用户等。

## Kibana

每个 Amazon ES 域默认安装 Kibana，但是Kibana 本身不支持 IAM 用户和角色，可通过一台EC2代理访问来控制访问限制。也可以使用Amazon Cognito 进行身份验证，但是宁夏区还没有看到有支持。

## Index 管理

- 索引状态管理 (ISM)

  可以定义自定义管理策略来自动执行日常任务，并将其应用于索引和索引模式。不过，Elasticsearch 6.6+才支持。这些策略有根据索引的存在时间、文档数、大小做滚动、删除、移动等操作（称之为生命周期管理）。

- 温存储（UltraWarm）

  在标准的ES中定义了热节点、温节点或冷节点等数据存储方式。在AWS中，使用了S3存储温数据。
  
- 模板（templates）

  通过定制一个模板，指定匹配规则，让Index自动应用一些规则，如：分片数、副本数、字段约定，尤其是索引的生命周期管理策略（policy）。比如指定setting中的"opendistro.index_state_management.policy_id": "..."。
  AWS没有提供模板管理工具，只能通过http API进行操作（put/delete/get _template/xxx）。

## 附：源数据加载

- Filebeat组件 
可以直接选择ES作为Filebeat的目的地。一般来说Filebeat是向Logstash、Kafka等输出数据。
  
- Kinesis Data Firehose

  Kinesis Data Firehose 支持 Amazon ES 作为传输目标。

- Logstash

  Amazon ES 支持两个 Logstash 输出插件：标准 Elasticsearch 插件和 logstash-output-amazon-es 插件。

- S3

- Kinesis Data Streams 

- Amazon DynamoDB

- CloudWatch Logs

- AWS IoT