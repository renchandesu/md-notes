# Elasticsearch基础

https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html

https://www.bilibili.com/video/BV1TR4y1Q7kd

### 简介

分布式文档存储中心，存储的是可序列化的JSON文档。天生支持分布式的搜索、聚合分析和存储引擎。是实现搜索引擎核心功能的技术之一。

- 分布式数据库
- OLAP系统
- 全文检索引擎

当一个文档被存储时，它会被索引，并在1秒内几乎实时地完全可搜索。Elasticsearch使用一种称为倒排索引的数据结构，能够支持非常快速的全文搜索。

### 基础配置

`cluster.name`集群的名称

`node.name`集群中节点的名称

`path.data`

`path.logs`

`bootstrap.memory_lock`内存锁

`network.host:`对外提供服务的ip地址 默认是本地地址或者配置成局域网的ip地址

`http.port`端口号

`transport.port`集群通信的端口

`xpack.security.enabled: true` 开启安全特点

```
xpack.security.http.ssl:

  enabled: true 开启https
```

生产模式下，会触发引导检查，对配置的一些不合理之处进行报错，阻止es的启动，而开发模式是没有这种检查的。如果你配置了集群的相关配置，那么则会触发生产模式。

es在首次启动时，会生成es的用户名密码，可以在之后进行修改

`./elasticsearch-reset-password --url http://localhost:9200 --username elastic -i`

### ES的近实时性

当一个文档被写入lucene后是不能立即被查询到的，es使用refresh执行刷新操作，调用lucence的reopen命令为内存中新写入的数据生成一个新的segment，此时被处理的数据才可以被检索到。
refresh操作的时间`refresh_interval` 默认1s 也可以调用显式refresh API或者在写入时带上refresh

```
PUT /test/_doc/2?refresh=true
```

### ES数据可靠性

##### 引入translog

当—个文档写入Lucence后是存储在内存中的，即使执行了refresh操作仍然是在文件系统缓存中，如果此时服务器宕机，那么这部分数据将会丢失。
为此Es增加了translog，当进行文档写操作时会先将文档写入Lucene，然后写入一份到translog，写入translog是落盘的
(如果对可靠性要求不是很高，也可以设置异步落盘，可以提高性能，由配置findex.translog.durability和index.translog.sync_interval)控制)，这样就可以防止服务器宕机后数据的丢失。由于translog是追加写入，因此性能比较好。与传统的分布式系统不同，这里是先写入Lucene再写入translog，原因是写入Lucene可能会失败，为了减少写入失败回滚的复杂度，因此先写入Lucene.

##### flush操作

另外每30分钟或当translog达到一定大小(由index.translog.flush_threshold_size 控制，默认51mb),
E5会触发一次flush操作，此时ES会先执行refresh操作将buffer中的数据生成segment，
然后调用lucene的commit方法将所有内存中的segment fsync到磁盘。
此时lucene中的数据就完成了持久化，会清空translog中的数据(6.x版本为了实现sequenceIlDs,不册除translog)

##### merge操作

由于refresh的间隔时间是1s，因此时间长了会产生大量的小segment，为此es运行一个任务检测当前磁盘中的segment，对符合条件的sement进行合并操作
这样可以减少segment的数量，提高查询速度，降低符合。
**而且，merge过程也是文档被执行更新或删除后，旧文档真正被删除的时机**
我们也可以手动去触发merge

##### 多副本机制

### 主要概念

#### 节点

一个es实例

##### 节点的角色

- 主节点： 一个集群只有一个，主要作用是对集群的管理
- 候选节点：当主节点故障时，参与选举，称为主节点
- 数据节点：保存包含已编入索引的文档的分片，处理数据相关操作（普通节点）
- 预处理节点（ingest）:也叫ingest pipeline 对数据写入前的预处理、如过滤等操作

##### 如何配置角色

`node.roles: [1,2,3,...]`默认是具备所有角色

#### 集群

ES 默认就是集群状态，整个集群是一份完整、互备的数据。

集群是一个或多个节点(服务器)的集合。集群中的节点一起存储数据，对外提供搜索功能。集群由一个唯一的名称标识，节点通过集群名称加入集群

##### 核心配置

- `network.host` 提供服务的ip地址，一般配置为本节点的内网地址，配置了此选项会触发引导检查
- `network.publish_host` 一般配置为公网地址，当集群的节点来自不同网络时配置
- `discovery.seed_host`配置候选节点的地址列表
- `cluster.initial_master_nodes`初次选举汇总用到的候选节点

##### 健康值状态检查

- 绿色 所有分片都可用
- 黄色 至少有一个副本不可用 但是所有的主分片都可用，此时集群能提供完整的读写，但是可用性降低
- 红色 至少有一个主分片不可用，数据不完整

`GET _cat|cluster/health?v 查看健康状态`

#### 索引

可以类比于MySQL中的表（7.x以后版本）

索引在创建成功后，有三个属性不可修改：

- 名字
- 主分片数量
- 字段类型

##### 组成

- alias 索引的别名
- settings 索引的设置，分片的数量、副本数量等
- Mapping 映射，定义了索引中包含了那些字段，以及字段的类型、长度、分词器等

##### 规范

1. 字母全小写
2. 多个单词用_连接

在es中，索引可能的含义有三种

- 表示源文件数据：即数据的载体
- 表示索引文件：用于加速检索而设计的数据文件。倒排索引、正排索引
- 表示创建数据的动作：即创建一个doc到某个索引

#### 映射 Mapping

Mapping用于描述字段，包含了一些属性，比如字段名称、类型、字段使用的分词器、是否评分、是否创建索引等属性，并且在 ES 中一个字段可以有多个类型。
比如说使用fields属性，让一个字段有多种不同的类型。

```
      "id":{ 
        "type": "long",
        "fields": {
          "time":{  //当使用id.time搜索时，为date类型的搜索
            "type":"date",
            "format":"yyyyMMddhhmmss"
          }
        }
      }
```

即使不进行手动映射，es也会对写入的文档进行自动类型推断，但是在生产环境，避免使用

主要有以下数据类型，并分为可以被分词的数据类型（text等）和不可以被分词的数据类型

##### 基本数据类型

- Numbers：数字类型，包含很多具体的基本数据类型
- binary：编码为 Base64 字符串的二进制值。
- boolean：即布尔类型，接受 true 和 false。
- alias：字段别名。
- Keywords：包含 keyword ★、constant_keyword 和 wildcard。
- Dates：日期类型，包括 data ★ 和 data_nanos，两种类型

date类型通过指定`"format":""`来设置格式.

Elasticsearch 内部会把日期转换为 UTC（世界标准时间），并将其存储为表示 milliseconds-since-the-epoch 的长整型数

```
      "time":{
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      }      
```

##### 对象关系类型（复杂类型）

- object：非基本数据类型之外，默认的 json 对象为 object 类型。
- nested ★：嵌套类型。 将数组中的每一个对象，当成一个独立的个体进行索引,避免使用object类型时出现扁平化现象，而影响检索的正确性。但是开销会更大，所以确保只在需要的时候使用
![[Pasted image 20230825111234.png]]
##### 结构化类型

- Range：范围类型，比如 long_range，double_range，data_range 等
- ip：ipv4 或 ipv6 地址
- version：版本号
- murmur3：计算和存储值的散列 

在插入range类型时，可以通过gte等字段设置范围
![img.png](img.png)

```
PUT my_index/my_type/1
{
  "expected_attendees": {
    "gte": 10,
    "lte": 20
  },
  "time_frame": {
    "gte": "2019-10-31 12:00:00",
    "lte": "2019-11-01"
  }
}
```

##### 聚合数据类型

- aggregate_metric_double：
- histogram：

4.2.5 文本搜索字段
text ★：文本数据类型，用于全文检索。text 类型的字段不用于排序，且很少用于聚合

在创建时，通过指定index选项来指定该字段是否可被索引 | "analyzer": "standard"指定分词器

##### 空间数据类型 ★

geo_point：纬度和经度点。
geo_shape：复杂的形状，例如多边形。
point：任意笛卡尔点。
shape：任意笛卡尔几何。

![img_1.png](img_1.png)

##### 其他类型

percolator：用Query DSL 编写的索引查询。

##### 文档排名类型

dense_vector：记录浮点值的密集向量。
rank_feature：记录数字特征以提高查询时的命中率。
rank_features：记录数字特征以提高查询时的命中率。

#### 类型

7.x以后不使用

就是为每个索引定义了不同类型的元数据，比如doc的数据格式等

#### 文档

就是关系型数据库中表中每一行 JSON数据

##### 文档的基本结构

- 元数据 `index`索引名称 `_id`文档id `_version`版本号
- `_source`存储的数据

#### 分片与副本

索引内文档保存的分区，分为主分片与从分片（副本）

主分片可读可写 副本只读 主分片数量一旦确定不可修改，副本数量可以修改

主分片和它的副本节点 不能同时存在于同一个节点上，且完全相同的副本不能同时存在于一个节点上

可以提高性能、可用性、扩展性

#### 分词器

把全文本转化为一系列单词的过程

由三种组件构成： 字符过滤器(预处理)，分词器、Token过滤器(将切分的单词进行加工)

##### 文档归一化

也就是对分词后的词项进行一定的处理，比如同义词、停用词、大小写统一等，从而**提高召回率和查询效率**
通过设置normalizer来指定

##### 分词器

##### 字符过滤器

##### toekn过滤器

##### 自定义分词器

### 索引的创建与修改

```
PUT <index_name>/_mapping 
{
     "settings": {
        "number_of_replicas": 1, 副本数量
        "number_of_shards": 5, 主分片数量
        "refresh_interval": 1000,  索引写入刷新的间隔
        "max_result_window": 10000 每次查询最多的返回数目 【涉及深度分页】
      },
      {
      "mappings": { 类型映射
        "properties": {
          "field_a": {
            "<parameter_name>": "<parameter_value>"
          },
}
```

### 文档的CRUD

#### 文档的新增

- 写入单条数据
  
  ```
  PUT <index>/_doc/<id>[?op_type=create] 如果已存在会失败 如果type是index则存在时全量更新
  {
   data
  }
  ```

POST kibana_sample_data_logs/_doc 不自己指定id的情况的写入，es会自动生成一个id
{
    data
}

#### 文档的更新

POST /<index>/_update/<_id> 局部更新
{
    data
}

#### 文档的部分更新

```
lucene支持对文档的整体更新, ES为了支持局部更新，在Lucene的Store索引中存储了一个_source字段，
该字段的key值是文档ID，内容是文档的原文。
当进行更新操作时先从_source中获取原文，与更新部分合并后，再调用lucene API进行全量更新, 
对于写入了ES但是还没有refresh的文档，可以从translog中获取。
另外为了防止读取文档过程后执行更新前有其他线程修改了文档，ES增加了版本机制，
当执行更新操作时发现当前文档的版本与预期不符，则会重新获取文档再更新。 
```

#### 文档的删除

DELETE /<index>/_doc/<_id>

#### 批量的增删改 _bulk
##### 批量写入

POST /kibana_sample_data_logs/_docs/_bulk 下面的json体要成对出现
{"index":{}} 这里可以指定id "_id"不指定默认
{data body}
{"index":{}}
{data body}
...

##### 批量更新

POST /kibana_sample_data_logs/_doc/_bulk
{"update":{"_id":"1111111222222"}}
{"doc":{"ip":"192.168.2.4"}}
...

##### 批量删除

POST /<index>/_doc/_bulk
{"delete":{"_id":"1111111222222"}}

#### Query DSL

##### 查询索引下文档总数

GET /_cat/count/<index1,index2>?v
GET /articles/_count doc数量

##### 空搜索

GET /_search 返回集群中所有索引的所有文档
GET /<index>/_search
GET /syslog*/_search 支持通配符
GET /_all/type1/_search 返回所有索引中指定类型的文档

```
*返回结果的解释*

![image-20230713200315305](/Users/renchan/Library/Application Support/typora-user-images/image-20230713200315305.png)

![image-20230713200533497](/Users/renchan/Library/Application Support/typora-user-images/image-20230713200533497.png)

##### 分页查询 
```

GET /<index>/_search?size=20&from=10 参数为size、from

```
**应该避免分页太深或者一次返回数据太多** 在分布式系统中，排序成本随着分页的深度的增加而成倍增加

##### DSL

使用字符串拼接的方式，我们也可以进行数据的检索，但是比较麻烦，把请求放在请求体中可以增加可读性

###### match_all

查询全部，如果带参数，当参数被匹配可以增加score.

与之相反的match_none不匹配任何文档
```

GET /index/_search
{
  "query": {
    "match_all": { "boost" : 1.2 } //这里可以是一个空{}
  }
}

```
###### match

对准确值字段使用match必须精确匹配

否则会使用正确的分析器去分析字符串
```

GET /kibana_sample_data_logs/_search
{
  "query":{
    "match": {
      "agent":"Mozilla/5.0 (X11; Linux x86_64; rv:6.0a1) Gecko/20110421 Firefox/6.0a"
    }
  }
}

```
###### match_phrase

将查询字符串解析成一个词项列表，只保留包含了所有词项的文档

###### Multi_match

在多个字段上执行相同的query
```

GET /<index>/_search
{
  "query": {
     "multi_match":{
    "query": "deb", 
    "fields":["agent","*_extension"]
  }
  }
}

```
###### range

范围查询

可以对日期、数字等类型使用

> Range queries on [`text`](https://www.elastic.co/guide/en/elasticsearch/reference/current/text.html) or [`keyword`](https://www.elastic.co/guide/en/elasticsearch/reference/current/keyword.html) fields will not be executed if [`search.allow_expensive_queries`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html#query-dsl-allow-expensive-queries) is set to false.
```

GET /kibana_sample_data_logs/_search
{
  "query": {
      "range":{
        "bytes": {
          "gte": 1999,
          "lte": 2000,
          "format": "yyyy-mm-dd"将时间转换为这种格式比较
        }

      }

  }
}

```
###### term

词条查询

用于确切值匹配,对查询条件不会分词，必须精准匹配

**注意：对text等全文类型使用term搜索是无法匹配的，除非使用.keyword指定为keyword类型**

###### terms

多个匹配条件，满足一个即返回
```

GET /index/_search
{
  "query": {
      "terms":{
        "referer": ["http://twitter.com/success/wendy-lawrence","http://www.elastic-elastic-elastic.com/success/james-mcdivitt"]
      }
  }
}

##### ids

```


根据id列表查询

```json
GET /_search
{
  "query": {
    "ids" : {
      "values" : ["1", "4", "100"]
    }
  }
}
```

##### exists

指定的字段必须具有可被索引的值,最常见的情况是字段为null

> 在一些情况下字段的值是不可被索引的
> 
> - The field in the source JSON is `null` or `[]`
> - The field has `"index" : false` set in the mapping
> - The length of the field value exceeded an `ignore_above` setting in the mapping
> - The field value was malformed and `ignore_malformed` was defined in the mapping

```
GET /_search
{
  "query": {
    "exists": {
      "field": "user"
    }
  }
}
```



##### 组合多查询

- bool查询 可以组合多个查询条件

```
{"query":{
    "bool":{
        "must":[], 必须匹配的条件，计算相关度得分 ;内部的子查询是上面的一种
        "must_not":[], 必须不符合的条件，不计算相关度得分
        "should":[], 如果符合提高得分,不是必须要满足，但是如果没有must语句则要求至少一个条件满足，也可以通过minimum_should_match控制
        "minimum_should_match":2
        "filter":[] 使用 filter 查询的评分将被忽略，并且结果会被缓存
    }
}}
```

##### Constant_score

用于只需要一个filter而没有其他查询的情况，取代只有filter的布尔查询

```
{
    "constant_score":{
        "filter":{
            //bool语句 或者其他条件语句
        }
    }
}
```

##### 精度控制

- 在text类型匹配时，只匹配所有词项全部包含的，或者至少需要匹配一定数量

```
{
    "query":{
        "match":{
            "title":{
                "query":"abc def",
                "operator":"and" //默认是or
                "minimum_should_match":4/"25%",
                "boost":1.2 //提升权重
            }

        }
    }
}
```

- 之前有提到过multi-match时，一个query搜索多个字段，可以通过`^`来提高某个字段的权重

```
{
    "query":{
        "multi_match":{
      "query":"abc def",
      "fields":["a1","a2^2"]
        }
    }
}
```

##### 子查询嵌套

查询是可以嵌套的，对于一个should、filter、match等，都可以通过bool查询进行嵌套

```
GET goods_en/_search
{
  "query": {
    "bool": {
      "must_not": [
        {
          "match": {
            "name": "iphone"
          }
        }
      ],
      "should": [
        {
          "bool": {
            "must": [
              {
                "match": {
                  "name": "phone"
                }
              }]
            }
          }
       ]
}
```

#### 聚合查询

在 Elasticsearch 中，聚合查询是一种分析和统计数据的功能。
聚合查询能够处理大量的数据，执行各种统计分析，如计算总数、平均值、最大值、最小值、标准差等等，并生成相应的报告。

text类型的数据不支持聚合

聚合查询包含以下部分：

- 查询条件：指定需要聚合的文档，可以使用标准的 Elasticsearch 查询语法，如 term、match、range 等等。
- 聚合函数：指定要执行的聚合操作，如 sum、avg、min、max、terms、date_histogram 等等。每个聚合命令都会生成一个聚合结果。
- 聚合嵌套：聚合命令可以嵌套，以便更细粒度地分析数据。

聚合函数的组成如下:

```
GET <index_name>/_search
{
  "aggs": {
    "<aggs_name>": { // 聚合名称需要自己定义
      "<agg_type>": {
        "field": "<field_name>"
      }
    }
  }
}
```

##### 桶查询

统计不同类型数据的数量

```
GET /kibana_sample_data_logs/_search
{
  "aggs": {
    "referer_count": {
      "terms": {
        "field": "referer"
      }
    }
  }
}
```

```
范围统计
GET /kibana_sample_data_logs/_search
{
  "aggs": {
    "referer_count": {
      "range": {
        "field": "bytes",
        "ranges":[
          {"to": 1000},{"to": 2000,"from": 1000},{"from": 2000}
          ]
      }
    }
  }
}
```

##### 指标聚合

统计最大值、最小值、平均值等

```
GET /kibana_sample_data_logs/_search
{
  "aggs": {
    "max_price": {
      "max": {
        "field": "bytes"
      }
    },
    "min_price": {
      "min": {
        "field": "bytes"
      }
    },
    "avg_price": {
      "avg": {
        "field": "bytes"
      }
    }
  }
}
```

##### 嵌套聚合

用于在某种聚合的计算结果之上再次聚合

```
GET /kibana_sample_data_logs/_search 这个查询的意思是对不同referer,都统计各个agent的个数,第二个聚合依赖于第一个
{
  "size": 0,
  "aggs": {
    "term_referers": {
      "terms": {
        "field": "referer"
      },
      "aggs":{
      "term_agents":{
        "terms": {
          "field": "agent.keyword"
          }
        }
      }
    }
  }
}
```

##### cardinality聚合

去重，cartinality metric，对每个bucket中的指定的field进行去重，取去重后的count，类似于count(distcint)  **cardinality，count(distinct)，5%的错误率，性能在100ms左右**

```
{
  "size" : 0,
  "aggs" : {
      "months" : {
        "date_histogram": {
          "field": "sold_date",
          "interval": "month"
        },
        "aggs": {
          "distinct_colors" : {
              "cardinality" : {
                "field" : "brand"
              }
          }
        }
      }
  }
}
```

##### 管道聚合 pipeline

管道聚合用于对聚合的结果进行二次聚合

管道聚合主要用来处理来自其他聚合的产出结果，而不是来自文档集的产出，并将信息添加到最终的输出结果中。

管道聚合可以引用它们执行计算所需的聚合，方法是使用 bucket_path 参数来指示到所需指标的路径。

管道聚合分为父级管道聚合以及兄弟管道聚合，具体的含义将通过实例来展示

- 兄弟级管道聚合：：在同一聚合级别上可以产生新的聚合。

- 父级管道聚合：由父聚合提供输出，子聚合能够产生新的桶，然后可以添加到父桶中。

###### bucket_path定义

> 聚合分隔符：“>”，指定父子聚合关系，如：“my_bucket>my_stats.avg”
> 权值分隔符： “.”，指定聚合的特定权值
> 聚合名称：<name of the aggregation>，直接指定聚合的名称
> 权值：<name of the metric>，直接指定权值
> 完整路径：agg_name[> agg_name]*[. metrics]，综合利用上面的方式指定完整路径
> 
> 特殊值：“_count”，输入的文档个数

###### Avg Bucket Aggregation

同级管道聚合，它计算同级聚合中指定度量的平均值。同级聚合必须是多桶聚合，针对的是度量聚合(metric Aggregation)。

类似的还有Max Bucket Aggregation Min Bucket Aggregation Sum Bucket Aggregation

Stats Bucket Aggregation

```
GET /kibana_sample_data_logs/_search
{
  "size": 0,
  "aggs": {
    "group_by_referer": {
      "terms": {
        "field": "referer"
      },
      "aggs": { 这个聚合是一个嵌套聚合
        "sums": {
          "sum": {
            "field": "bytes"
          }
        }
      }
    },
    "pipel": {
      "avg_bucket": { 这个就是平均值管道聚合，作用是计算sums的平均值
        "buckets_path": "group_by_referer>sums" 这个是找到对应聚合的路径
      }
    }
  }
}
```

###### Cumulative Sum Aggregation

一种累加，其父聚合必须是histogram

```
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "cumulative_sales": {
                    "cumulative_sum": {
                        "buckets_path": "sales" 
                    }
                }
            }
        }
    }
}

作者：中间件兴趣圈
链接：https://juejin.cn/post/6939332790311714824
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

###### Bucket Sort Aggregation

一种父管道聚合，它对其父多桶聚合的桶进行排序。并可以指定多个排序字段。每个bucket可以根据它的_key、_count或子聚合进行排序。此外，可以设置from和size的参数，以便截断结果桶。

语法如下：

```
{
    "bucket_sort": {
        "sort": [
            {"sort_field_1": {"order": "asc"}}, 这里的sort_field是指的聚合函数名字
            {"sort_field_2": {"order": "desc"}},
            "sort_field_3"
        ],
        "from": 1,用与对父聚合的桶进行截取，该值之前的所有桶将忽略，也就是不参与排序，默认为0。
        "size": 3
    }
}
```

- sort
   定义排序结构。
- from
   用与对父聚合的桶进行截取，该值之前的所有桶将忽略，也就是不参与排序，默认为0。
- size
   返回的桶数。默认为父聚合的所有桶。
- gap_policy
   当管道聚合遇到不存在的值，有点类似于term等聚合的(missing)时所采取的策略，可选择值为：skip、insert_zeros。
  - skip：此选项将丢失的数据视为bucket不存在。它将跳过桶并使用下一个可用值继续计算。
  - insert_zeros：默认使用0代替。

```http
GET /kibana_sample_data_logs/_search
{
  "size": 0,
  "aggs": {
    "group_by_referer": {
      "terms": {
        "field": "referer"
      },
      "aggs": {
        "sums": {
          "sum": {
            "field": "bytes"
          }
        },
        "bucket_rank": {
          "bucket_sort": {
            "sort": [
              {
                "sums": {
                  "order": "desc"
                }
              }
            ],
            "from": 1,
            "size": 2,
            "gap_policy": "skip"
          }
        }
      }
    }
  }
}
```

##### 分页和排序

###### 对聚合的结果进行排序

```
GET <index_name>/_search
{
  "size": 0, // 表示在查询结果中不现实源数据
  "aggs": {
    "term_lv": {
      "terms": {
        "field": "<doc_value_field>",
        "size": 10, // 对聚合结果取前 10 条数据
        "order": {
          "<order_type>"[_count:按照文档数量排序,_key:对聚合结果的key排序,_term]: "<order_value>"[asc,desc] 
        }
      }
    }
  }
}
```

还可以进行多字段排序 先..再..

```
GET goods/_search?size=0
{
  "aggs": {
    "first_sort": {
      "terms": {
        "field": "brand.keyword",
        "order": [
          {
            "_count": "desc"
          },
          {
            "_key": "asc"
          }
        ]
      }
    }
  }
}
```

还可以按照内部聚合的结果，对上层聚合进行排序

```
GET goods/_search
{
  "size": 0,
  "aggs": {
    "agg_term_tag": {
      "terms": {
        "field": "brand.keyword",
        "order": {
          "agg_stats_price.avg": "asc" 注意这里
        }
      },
      "aggs": {
        "agg_stats_price": {
          "stats": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

##### 过滤器

Filter 用于局部聚合查询条件过滤，可在指定聚合函数内嵌套使用，嵌套使用后会在聚合结果中包含符合filter条件的文档

```
POST <index_name>/_search
{
  "aggs": {
    "phone_agg": {
      "filter": { 过滤器
          "terms": {
              "type.keyword": [
                "耳机",
                "手机",
                "电视"
              ]
           }
      },
      "aggs": {
        ...
      }
    }
  }
}
```

post_filter 只过滤搜索结果，而对聚合结果没有影响

```
GET goods/_search
{
  "aggs": {
    "tags_bucket": {
      "terms": {
        "field": "tags.keyword"
      }
    }
  },
  "post_filter": {
    "term": {
      "tags.keyword": "IOS"
    }
  }
}
```

#### Filters过滤

Filters Aggregation 是多过滤器聚合，可以把符合多个过滤条件的文档分到不同的桶中，即每个分组关联一个过滤条件，并收集所有满足自身过滤条件的文档。

```
{
  "size": 0,
  "aggs": {
    "messages": {
      "filters": {
        "filters": {
          "errors": { "match": { "body": "error" } },
          "warnings": { "match": { "body": "warning" } }
        }
      }
    }
  }
}
```

##### 全局聚合过滤

query可以和aggs一起使用，aggs使用的文档是query的结果

```
POST goods/_search?filter_path=aggregations
{
  "size": 0,
  "query": {
    "term": {
      "type.keyword": "phone"
    }
  },
  "aggs": {
    "avg_price": {
      "avg": {
        "field": "price"
      }
    }
  }
}
```

```
GET goods/_search
{
  "size": 10,
  "query": {
    ...
  },
  "aggs": {
    "avg_price": {
      ...
    },
    "avg_price": {
      "global": {}, 加上这个，表明这个聚合使用的是全部的文档
      "aggs": {
        ...
      }
    }
  }
}
```

##### 对聚合结果的查询 top hits

```
GET /kibana_sample_data_logs/_search
{
  "size": 0,
  "aggs": {
    "term_referers": {
      "terms": {
        "field": "referer"
      },
      "aggs": {
        "top_aggs": {
          "top_hits": {
            "size": 10, 数量
            "sort": [
              {
                "bytes": {"order": "desc"}
              }  
            ],
            "from": 90 从哪里开始 与size 相加不能超过100
          }
        }
      }
    }  
  }
}
```

#### histogram

用于区间统计

```
GET /kibana_sample_data_logs/_search
{
  "size": 0, 
  "aggs": {
    "h_n": {
      "histogram": {
        "field": "bytes",             
        "interval": 100,        #区间间隔    
        "keyed": true,            #返回数据的结构化类型    
        "min_doc_count": 10,    #返回桶的最小文档数阈值，即文档数小于num的桶不会被输出
        "missing": 10                #空值的替换值，即如果文档对应字段的值为空输出
      }
    }
  }
}
```

#### date histogram

```
GET /kibana_sample_data_logs/_search
{
  "size": 0, 
  "aggs": {
    "h_n": {
      "date_histogram": {
        "field": "timestamp",             
        "interval": "1D", 日期的间隔 D M 
        "format": "yyyy-MM-dd"
      }
    }
  }
}
```

- interval：时间间隔的参数可选项
  - fixed_interval：ms（毫秒）、s（秒）、 m（分钟）、h（小时）、d（天），注意单位需要带上具体的数值，如2d为两天。需要当心当单位过小，会导致输出桶过多而导致服务崩溃。
  - calendar_interval： year quarter month week day hour minute
- extended_bounds：指定自定义的时间范围。可以通过提供"min"和"max"参数来限制聚合操作的时间范围。
- keyed：指定是否将结果按时间段的键值对形式返回。如果设置为true，每个时间段的结果将以键值对的形式返回，默认为false。

**使用时的坑：**

date_histogram返回的桶前后时间会有偏差
我的查询返回时1573113999000到1573114599000，也就是2019-11-07 16:06:39到2019-11-07 16:16:39，以10s分割，但是返回的第一个桶的时间是2019-11-07 16:06:30，已经超过了我 extended_bounds设置的时间。至于偏差多少时间，跟interval有关，设置1s就不会偏差

**原因：**

es内部计算桶的时间跟我们理解的不一样，不是开始时间 + interval。
计算es桶聚合bucket_key的公式是 bucket_key = Math.floor((value - offset) / interval) * interval + offset
根据公式console.log(Math.floor((1573113999000 - 0) / 10000) * 10000 + 0)
结果是1573113990000，也就是2019-11-07 16:06:30


#### percentile

用于评估当前数值分布情况，比如 99 percentile 是 1000 ， 是指 99%的数值都在 1000 以内。

常见的一个场景就是我们制定 SLA 的时候常说 99% 的请求延迟都在100ms 以内，这个时候你就可以用 99 percentile 来查一下

```
GET <index_name>/_search?size=0
{
  "aggs": {
    "<percentiles_name>": {
      "percentiles": {
        "field": "price",
        "percents": [
                  percent1，                #区间的数值，如5、10、30、50、99 即代表5%、10%、30%、50%、99%的数值分布
                  percent2，
                  ...
        ]
      }
    }
  }
}
```

#### 邻接矩阵

Elasticsearch中的Adjacency Matrix聚合是一种强大的聚合类型，它可以用于分析和发现数据中的关系和连接。
Adjacency Matrix聚合在许多使用场景中都非常有用。

#### DSL（一些查询的草稿）

```HTTP
GET _search[/?size=10] 查询全部索引的文档，默认查10条
GET /articles[/_search[?from=9&size=100&timeout=10]] 没有后面参数获取索引的元
GET /articles/_count doc数量
数据
PUT /test 创建索引
{
  "settings": {
    "number_of_replicas": 1,
    "number_of_shards": 5,
    "refresh_interval": 1000,  索引写入刷新的间隔
    "max_result_window": 10000 每次查询最多的返回数目 【涉及深度分页】
  },
  {
  "mappings": {
    "properties": {
      "field_a": {
        "<parameter_name>": "<parameter_value>"
      },
      ...
    }
  }
}_cat
}
PUT /test/_settings 修改索引settings
{
  "settings": {
    "number_of_replicas": 1,
    "number_of_shards": 5 这个没法改【静态设置 只有在索引创建或关闭时设置】
  }
}
DELETE /articles 删除索引
GET /_cat/indices?v 查看所有索引的信息
GET /_cat/indices
GET /index_name/_mapping 查看索引的映射信息
POST /_reindex 重建索引内的文档，如果索引内没有文档无法创建成功，并且不拷贝索引的结构
{
    "dest":{"index":""},
    "source":{"index":""}
}
POST /articles/article/3 写入文档
{
  "id":3,
  "title":"黑暗dd",
  "author":"renchan",
  "content":"ok sfj",
  "created_date":"2022-11-12"

}

# 更新文档
POST /articles/article/1/_update
{
  "doc": {
    "title":"asd"
  }
}


# 查询所有
GET /articles/article/_search
{
  "query": {

    "match_all": {}
  }
}

# 基于关键词查询 注意类型是否分词
GET /articles/article/_search
{
  "query": {

    "term": {
      "conteng": {
        "value": ""
      }
    },
    "ids": {
      "values": []
    }
  }
}

# 范围查询
GET /articles/article/_search
{
  "query": {
    "range": {
      "FIELD": {
        "gte": 10,
        "lte": 20
      }
    }
  }
}

# 前缀查询 即对每个分词的前缀进行匹配
GET /articles/article/_search
{
  "query": {
    "prefix": {
      "content": {
        "value": "s"
      }
    }
  }
}


# 通配符
GET /articles/article/_search
{
  "query": {
    "wildcard": {
      "content": {
        "value": "*s*"
      }
    }
  }
}

# fuzzy 最大模糊次数0-2
# 次数与关键词长度有关
GET /articles/article/_search
{
  "query": {
    "fuzzy": {
      "author": "rechan"
    }
  },"highlight": {
    "pre_tags": {},
    "post_tags": {},
    "require_field_match": "true",
    "fields": {
      "*":{}
    }
  },"sort": [
    {
      "FIELD": {
        "order": "desc"
      }
    }
  ],"size": 20,"from": 0,
  "_source":[] 返回指定字段
}


# 布尔查询 多条件查询
# must 都必须满足 should 满足一个即可
GET /articles/article/_search
{
  "query": {
    "bool": {
      "must": [
        {

        }
      ],"should": [
        {}
      ],"must_not": [
        {}
      ]
    }
  }
}

# 多字段查询
# 如果字段类型支持分词，则会对query进行分词否则不分词
GET /articles/article/_search
{
  "query": {
    "multi_match": {
      "query": "",
      "fields": []
    }

  }
}


# 过滤查询
# 过滤会筛选出符合条件的数据，不会计算得分，而且可以缓存文档，单从性能考虑，过滤操作性能更高，所以对于大量的数据，可以先进行过滤初筛，再精准查询
# 必须与布尔查询配合使用


GET /articles/article/_search
{
"query": {
  "bool": {
    "must": [
      {"term": {
        "author": 
          "value": "renchan"
        }
      }}
    ],"filter": {"ids": {
      "values": [
        1,2
      ]
    }}
  }
}
}
```

#### 高亮显示

```html
POST /sms-logs-index/_search
{
  "query": {
    "match": {
      "smsContent": "魅力"
    }
  },
  "highlight": {
    "fields": {
      "smsContent": {}
    },
    "pre_tags": "<font color='red'>",
    "post_tags": "</font>",
    "fragment_size": 10
  }
}
```

### 整合Springboot

#### ElasticsearchTemplate 增删改查

```java
import com.alibaba.fastjson.JSON;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.io.IOUtils;
import org.apache.commons.lang3.StringUtils;
import org.elasticsearch.index.query.BoolQueryBuilder;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.index.query.TermQueryBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.Resource;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.data.elasticsearch.core.ElasticsearchRestTemplate;
import org.springframework.data.elasticsearch.core.SearchHit;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.data.elasticsearch.core.document.Document;
import org.springframework.data.elasticsearch.core.mapping.IndexCoordinates;
import org.springframework.data.elasticsearch.core.query.AliasQuery;
import org.springframework.data.elasticsearch.core.query.IndexQuery;
import org.springframework.data.elasticsearch.core.query.NativeSearchQuery;
import org.springframework.data.elasticsearch.core.query.UpdateQuery;
import org.springframework.stereotype.Component;
import java.io.IOException;
import java.nio.charset.Charset;
import java.util.ArrayList;
import java.util.List;

@Slf4j
@Component
public class CommonESRepository {


    /**
     * 索引的setting
    */
    @Value("classpath:json/es-setting.json")
    private Resource esSetting;

    private final ElasticsearchRestTemplate elasticsearchRestTemplate;

    public CommonESRepository(ElasticsearchRestTemplate elasticsearchRestTemplate) {
        this.elasticsearchRestTemplate = elasticsearchRestTemplate;
    }

    /**
     * 判断索引是否存在
     * @param indexName 索引名称
     * @return boolean
     */
    public boolean indexExist(String indexName){
        if(StringUtils.isBlank(indexName)){
            return false;
        }
        IndexCoordinates indexCoordinates = IndexCoordinates.of(indexName);
        return elasticsearchRestTemplate.indexOps(indexCoordinates).exists();
    }

    /**
     * 判断index是否存在 不存在创建index
     * @param index 索引实体
     * @param indexName 创建索引的名称
     */
    public void indexCreate(Class index,String indexName){
        if(null == index || StringUtils.isBlank(indexName)){
            return;
        }
        IndexCoordinates indexCoordinates = IndexCoordinates.of(indexName);
        if(!elasticsearchRestTemplate.indexOps(indexCoordinates).exists()){
            // 根据索引实体，获取mapping字段
            Document mapping = elasticsearchRestTemplate.indexOps(indexCoordinates).createMapping(index);
            // 创建索引
            String esSettingStr = null;
            try {
                // 读取setting配置文件
                esSettingStr = IOUtils.toString(esSetting.getInputStream(), Charset.forName("utf-8"));
            } catch (IOException e) {
                log.error("读取setting配置文件错误", e);
            }
            // setting
            Document setting = Document.parse(esSettingStr);
            elasticsearchRestTemplate.indexOps(indexCoordinates).create(setting);
            // 创建索引mapping
            elasticsearchRestTemplate.indexOps(indexCoordinates).putMapping(mapping);
        }
    }

    /**
     * 根据索引名称，删除索引
     * @param index 索引类
     */
    public void indexDelete(String index){
        elasticsearchRestTemplate.indexOps(IndexCoordinates.of(index)).delete();
    }


    /**
     * 索引添加别名
     * @param indexName 索引名
     * @param aliasName 别名
     */
    public boolean indexAddAlias(String indexName,String aliasName){
        if(StringUtils.isBlank(indexName) || StringUtils.isBlank(aliasName)){
            return false;
        }
        // 索引封装类
        IndexCoordinates indexCoordinates = IndexCoordinates.of(indexName);
        // 判断索引是否存在
        if(elasticsearchRestTemplate.indexOps(indexCoordinates).exists()){
            // 索引别名
            AliasQuery query = new AliasQuery(aliasName);
            // 添加索引别名
            boolean bool = elasticsearchRestTemplate.indexOps(indexCoordinates).addAlias(query);
            return bool;
        }
        return false;
    }

    /**
     * 索引别名删除
     * @param indexName 索引名
     * @param aliasName 别名
     */
    public boolean indexRemoveAlias(String indexName,String aliasName){
        if(StringUtils.isBlank(indexName) || StringUtils.isBlank(aliasName)){
            return false;
        }
        // 索引封装类
        IndexCoordinates indexCoordinates = IndexCoordinates.of(indexName);
        // 判断索引是否存在
        if(elasticsearchRestTemplate.indexOps(indexCoordinates).exists()){
            // 索引别名
            AliasQuery query = new AliasQuery(aliasName);
            // 删除索引别名
            boolean bool = elasticsearchRestTemplate.indexOps(indexCoordinates).removeAlias(query);
            return bool;
        }
        return false;
    }

    /**
     * 索引新增数据
     * @param t 索引类
     * @param <T> 索引类
     */
    public <T> void save(T t){
        // 根据索引实体名新增数据
        elasticsearchRestTemplate.save(t);
    }

    /**
     * 批量插入数据
     * @param queries 数据
     * @param index 索引名称
     */
    public void bulkIndex(List<IndexQuery> queries, String index){
        // 索引封装类
        IndexCoordinates indexCoordinates = IndexCoordinates.of(index);
        // 批量新增数据，此处数据，不要超过100m，100m是es批量新增的筏值，修改可能会影响性能
        elasticsearchRestTemplate.bulkIndex(queries,indexCoordinates);
    }

    /**
     * 根据条件删除对应索引名称的数据
     * @param c 索引类对象
     * @param filedName 索引中字段
     * @param val 删除条件
     * @param index 索引名
     */
    public void delete(Class c,String filedName,Object val,String index){
        // 匹配文件查询
        TermQueryBuilder termQueryBuilder = QueryBuilders.termQuery(filedName, val);
        NativeSearchQuery nativeSearchQuery = new NativeSearchQuery(termQueryBuilder);
        // 删除索引数据
        elasticsearchRestTemplate.delete(nativeSearchQuery,c, IndexCoordinates.of(index));
    }

    /**
     * 根据数据id删除索引
     * @param id 索引id
     * @param index
     */
    public void deleteById(Object id,String index){
        if(null != id && StringUtils.isNotBlank(index)){
            // 根据索引删除索引id数据
            elasticsearchRestTemplate.delete(id.toString(),IndexCoordinates.of(index));
        }
    }

    /**
     * 根据id更新索引数据,不存在则创建索引
     * @param t 索引实体
     * @param id 主键
     * @param index 索引名称
     * @param <T> 索引实体
     */
    public <T> void update(T t,Integer id,String index){
        // 查询索引中数据是否存在
        Object data = elasticsearchRestTemplate.get(id.toString(), t.getClass(), IndexCoordinates.of(index));
        if(data != null){
            // 存在则更新
            UpdateQuery build = UpdateQuery.builder(id.toString()).withDocument(Document.parse(JSON.toJSONString(t))).build();
            elasticsearchRestTemplate.update(build,IndexCoordinates.of(index));
        }else {
            // 不存在则创建
            elasticsearchRestTemplate.save(t);
        }
    }

    /**
     * 查询数据根据实际情况自行修改
     * @param c
     * @param search
     * @param index
     * @param <T>
     * @return
     */
    public <T> List<T> getList(Class<T> c,String search,String index){
        TermQueryBuilder termQueryBuilder = QueryBuilders.termQuery("title", search);
        NativeSearchQuery nativeSearchQuery = new NativeSearchQuery(termQueryBuilder);
        nativeSearchQuery.addSort(Sort.by(Sort.Direction.DESC,"_score"));
        nativeSearchQuery.setTrackTotalHits(true);
        SearchHits<T> tax_knowledge_matter = elasticsearchRestTemplate.search(nativeSearchQuery, c, IndexCoordinates.of(index));
        List<SearchHit<T>> searchHits = tax_knowledge_matter.getSearchHits();
        List<T> returnResult = new ArrayList<>();
        searchHits.forEach(item -> {
            T content = item.getContent();
            returnResult.add(content);
        });
        return returnResult;
    }

    public <T> List<T> getList(Class<T> c,String search,int page,int size,Integer siteId,String index){
        BoolQueryBuilder title = QueryBuilders.boolQuery().
                must(QueryBuilders.matchQuery("title", search)).
                must(QueryBuilders.termQuery("siteId", siteId));
        NativeSearchQuery nativeSearchQuery = new NativeSearchQuery(title);
//        nativeSearchQuery.addSort(Sort.by(Sort.Direction.DESC,"_score"));
        nativeSearchQuery.setPageable(PageRequest.of(page,size,Sort.by(Sort.Direction.DESC,"_score")));
        nativeSearchQuery.setTrackTotalHits(true);
        SearchHits<T> tax_knowledge_matter = elasticsearchRestTemplate.search(nativeSearchQuery, c, IndexCoordinates.of(index));
        List<SearchHit<T>> searchHits = tax_knowledge_matter.getSearchHits();
        List<T> returnResult = new ArrayList<>();
        searchHits.forEach(item -> {
            T content = item.getContent();
            returnResult.add(content);
        });
        return returnResult;
    }
}
```
