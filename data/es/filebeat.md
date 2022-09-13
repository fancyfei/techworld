# filebeat

Filebeat是Elastic旗下Beats系列产品中的一员，轻量型日志采集器。Elastic就是那个最为广泛使用的ELK（Elasticsearch , Logstash, Kibana）日志采集分析方案的开源软件提供商。它采集日志的方式是监控指定的日志文件，然后将数据输出到指定的目的地（Elasticsearch 、Logstash、Kafka等），这种方案对上层应用无侵入，有很好的可扩展性。

## 架构组件

Filebeat有两个主要组件：inputs 和 harvester。harvester负责读取单个文件的内容。inputs 负责管理多个harvester并找到所有要读取的文件来源，input 同时也支持kafka、stdin、syslog等多种源。还有Output组件管理输出源；Pipeline管理缓存、输入、输出等；Registrar管理文件处理状态等。

Filebeat 保存每个文件的状态（读取的最后偏移量）并经常将状态刷新到磁盘上的注册文件中（data/registry），保证重启后能维持文件读取状态，可以保证日志At least once的上报。

Harvester将数据写入到pipeline中，数据将会写入缓存中，Output会从缓存将数据读走。整个生产消费的过程都是由Pipeline进行调度的。

在Filebeat中，还有大量的内置的Module，简化了公共日志格式的收集、解析和可视化，启动即可（./filebeat modules enable）。还可以自定义Module，

## 数据的采集

- 读取数据

  对多个需要监控的文件支持模糊匹配，还可以使用exclude_files参数进行排除，它支持正则匹配。

  Filebeat是逐行读取日志数据的，同时可以按规则合并多行，最后形成一条格式为Json的记录，在此记录中另外还包括了此次采集的所有信息，如系统信息、文件信息、时间戳等等。数据的多行合并使用了正则匹配，multiline参数中pattern、negate、match分别定义了正则、匹配条件、组合模式等。

- 输出格式化

  如需要将记录进行切分为细粒度的数据，除了使用Filebeat modules实现，就有两种常见方式，应用的日志定义为Json格式，就可以直接发送ES，对于不能更换日志格式的则可以使用Grok模块（Logstash、ES都支持此插件）。

  同时通过include_fields参数过滤为自已需要的字段。

## pipeline & grok

通过filebeat直接写数据到es中，要对日志内容做处理的话设置对应的pipeline就可以，使用pipeline，需要先在es中增加对应的pipeline。processor使用的grok，主要是patterns的编写，es的默认正则pattern可以直接使用。pipeline/_simulate可以测试编写的grok的正确性。最后在output.elasticsearch中指定pipeline。pipeline处理日期时需要注意的是时区 的处理，最好加上"timezone": "Asia/Shanghai"。

## 附：日志滚动处理

ES的索引管理中，有索引生命周期策略的管理，可对热、温、冷、删除等不同的生命周期进行控制。对日志等索引可以设置过n天即可删除等策略。此策略是应用在模板上的，所以需要索引创建时指定模板。

## 注：

- grok正则参考：https://github.com/logstash-plugins/logstash-patterns-core/blob/master/patterns/grok-patterns

- 大量日志需要采集时的优化，spool_size的默认值是2048，可能会引起OOM。