# Elasticsearch 索引滚动Rollover

使用Elasticsearch处理海量的日志记录的场景中，如果数据都放到相同的索引中，那么随着时间的堆积，后续数据的插入和查询的性能则会越来越差。Elasticsearch 从 5.0 开始，为此场景提供了rollover功能，当某个别名指向的实际索引过大的时候，自动将别名指向下一个实际索引。

## rollover index

可以给索引设置一定的条件，并提供一个对外使用的索引别名，当索引达到指定的条件时，就会按照一定的规则建立新的索引，对应的别名也会指向新的索引，后续的数据就会写到新建的索引中。由于外部应用使用的是索引的别名，对这种变化也是完全无感知的。

建立别名，在创建index时可以给定（由aliases参数指定），或者后期添加：

```
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "my-index",
        "alias": "logs"
      }
    }
  ]
}
```

对Index进行Roller请求，如果条件满足，自动创建并将别名指向下一个索引。

```
POST my-index/_rollover {
    "conditions": {
        "max_age":   "1d",
        "max_docs":  10000000
    }
}
```

条件可选的参数有：

| 参数名   | 说明                                               |
| -------- | -------------------------------------------------- |
| max_age  | 指定索引存在的最大周期，如1d,7d,1m,1y              |
| max_docs | 存放的最低索引文档数，这个不包括副本中的索引文档数 |
| max_size | 索引中主分片所占用的空间大小                       |

## 自动滚动

自动滚动有两种方案，一个是基于别名和生命周期，一个是专门用在时序数据的DataStream和生命周期。DataStream是7.9的新特性，可以认为是自动管理别名和索引的整体方案。当然也还有一种手工的办法，手工或定时脚本执行Roller请求。

- **生命周期**

管理索引滚动的核心是索引生命周期管理策略（Index Liftcycle Policy），一个典型的 ILM 通常有4个阶段：Hot, Warm, Cold, 以及 Delete，根据条件进行阶段的转移，条件则是上文提到的条件：max_age、max_docs、max_size。每个阶段对应很多操作，如rollover、shrink、readonly、delete等等

```json
PUT _ilm/policy/my_rollover_policy {
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "rollover": {
            "max_primary_shard_size": "3gb",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "5d",
        "actions": {
          "readonly": {},
          "allocate": {
            "number_of_replicas": 0
          }
        }
      },
      "delete": {
        "min_age": "15d",
        "actions": {
          "delete": {
            "delete_searchable_snapshot": true
          }
        }
      }
    }
  }
}
```

- **模板**

通过模板管理索引的策略与别名可以做到自动化。模板定义字段与分片之类参见API文档，这儿只提及与滚动相关配置。

```json
PUT _index_template/my-log {
  "template": {
    "settings": {
      "index": {
        "lifecycle": {
          "name": "my_log_rollover_policy",
          "rollover_alias": "my-log-index"
        }
      }
    },
    "mappings": {
      "dynamic_templates": [...],
      "properties": {...}
    }
  },
  "index_patterns": [
    "my-log*"
  ]
}

```

重点是lifecycle配置，指定生命周期策略，指定滚动别名。

创建索引时需要指定别名，必须与index.lifecycle.rollover_alias相同。

```json
PUT my-log-000001
{
    "aliases": {
        "my-log-index": {}
    }
}
```

- **基于别名的自动滚动**

经过设置生命周期，创建模板，创建带别名的索引，简单几步就可以完成整个配置，此索引达到指定条件时，便自动创建下一个索引，将别名指定到新的索引，名称则是固定前缀后加一个六位数的零填充整数。

- **基于DataStream的自动滚动**

与上面的方案类似，创建生命周期，创建模板，创建数据流，即可完成整个配置。需要注意的是，只有一点差别，就是创建模板时，不需要再指定index.lifecycle.rollover_alias，并加标识"data_stream": { }，表明自己是个DataStream即可。

```shell
PUT _data_stream/my-log
```

当发送数据到data stream时，达到一定条件，则会自动滚动。

所有的配置，都可以在Kabana面板上可视化操作。

> DataStream相当于自动化的完成别名的指向，索引的滚动则是由生命周期来定义。
>
> DataStream相当对这类滚动的时序数据类型的索引进行了统一的包装。
