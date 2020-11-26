# Elasticsearch Template（索引模板）

索引模板，是一种复用机制，当新建一个 Elasticsearch 索引时，自动匹配模板，完成索引的基础部分设置。通常用来解决多索引场景下统一字段、基础配置等。

## 统一设置项

- index_patterns为模板匹配规则，正则表达式。
- order为模板优先级(当索引同时匹配多个模板，先应用数值小的模板，再以数值大的模板配置进行覆盖)。
- settings指索引层面的设置，如分片数、副本数等。
- aliases指定索引的别名。
- mappings主要是一些说明信息，大致又分为\_all、\_source、properties这三部分。
- properties中指定特定类型，否则则会自动匹配为dynamic_templates中的text或keyword类型。

## 基础操作

- 新增 PUT _template/template_1{"index_patterns": [],"aliases" : {},"settings": {},"mappings": {"properties": {} } }
- 删除 DELETE _template/template_1
- 修改同新增，会自动覆盖，新模板只对新创建的索引生效，对历史索引不起作用。
- 查询 GET _template/template_1

## 动态映射

由于ES可以不预先定义字段类型，所以在处理传来的没有配置好的字段时，希望做一些通用配置，就使用到了mappings中dynamic_templates来做动态字段的映射。它能以源数据类型来匹配，也可以以源字段名正则匹配，然后重写为指定类型。有以下几个具体配置：

- match_mapping_type：被匹配的被重写的源数据类型。
- match/unmatch：正则匹配字段名。
- mapping：将重写的目的类型。

## 版本号说明

多个索引模板可能匹配一个索引，可以使用order属性为索引模板指定顺序。从顺序较小的开始寻找，order越大，越优先（前提是匹配index_patterns表达式），直到把所有的模板都处理过。最后的模板会覆盖之前的模板。order相同就不知道先后顺序了。

## 采用索引模板文件

Http的API可以创建模板，但是同样可以通过配置文件实现。放置在config路径下面的templates目录。创建即生效，不需要重启与reload，修改也同样立即生效覆盖。