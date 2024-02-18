# Elasticsearch 索引的生命周期 

## 前言

- 本文基于elasticsearch v6.8.3
- 主要包括两部分内容
  - 索引的生命周期有哪些
  - 索引的生命周期如何设置 有哪些操作
  - 实际实验，对于不同索引的生命周期，可以进行哪些操作

## 索引的生命周期

### 索引生命周期介绍

#### Phase

##### Open

打开索引意味着索引是可读写的状态，可以执行搜索、写入和其他操作。当索引是打开状态时，数据是可用的，并且可以接受新的写入操作。

##### Close

关闭的索引将大大减少对集群的负载，但是不能进行读写操作。一个关闭的索引可以再次被打开。

> To reduce the risk of data loss, avoid keeping closed indices in your cluster for long periods of time. When a node leaves your cluster, Elasticsearch does not rebuild the lost copies of the shards in any closed indices. As you replace the nodes in your cluster over time you will gradually lose the shard data for any closed indices.

> Closed indices are ignored by many APIs. For instance, the shards of a closed index are not included in the output of the [*cat shards*](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/cat-shards.html) API.

##### Hot

索引的读写非常的活跃

##### Warm

索引不再会被更新，但是仍然被查询

##### Cold

索引不再被更新，很少被查询

##### Delete

索引不再被需要，能够被删除

##### Frozen

可通过API将索引冻结，冻结的索引只支持读操作，不支持任何写入

#### Action

索引被创建后，会根据定义好的阶段，然后执行定义好的Actions。每个阶段的Actions必须执行结束才会进行下一个阶段；每个Phase有一个最小时间（min_age），只有时间超过了，才会进入下一个阶段。

需要注意，阶段的检查默认10min进行一次，如果需要更改（一般用于demo的学习）可以对如下参数进行设置

```
PUT _cluster/settings
{
    "transient": {
      "indices.lifecycle.poll_interval": "10s"
    }
}
```





## 实际操作

```
PUT _ilm/policy/test_policy 创建ILM策略
{
  "policy": {                       
    "phases": {
      "hot": {                      
        "actions": {
          "rollover": {             
            "max_size": "5GB",
            "max_age": "1m",
            "max_docs": 5
          }
        }
      }
    }
  }
}

PUT _template/datastream_template 创建一个索引template
{
  "index_patterns": ["logs-*"],   匹配索引名称              
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "index.lifecycle.name": "test_policy",      
    "index.lifecycle.rollover_alias": "logs"    别名
  }
}

PUT logs-000001 创建第一个索引， 将应用上述模板规则
{
  "aliases": {
    "logs": {
      "is_write_index": true
    }
  }
}

POST /logs/_doc 写入
{
  "name":"hello"
}

GET logs-*/_ilm/explain 查看索引生命周期的执行状态解释

结果：每10分钟进行一次检测，如果符合rollover条件，则进行rollover,创建一个新的索引，同时alais的write指向最新创建的索引

```



