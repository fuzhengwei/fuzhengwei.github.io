#### elasticsearch可以用来做什么
全文检索、结构化搜索、分析以及这三个功能的组合。

#### 如何于es进行交互
##### java代码
·使用java代码时可以使用es内置的客户端：Node client和transport client

·java客户端通过9300端口使用es原生传输协议和集群交互，集群中的各个节点也是通过9300端口进行交互。

##### RESTfull API
无论使用什么语言，我们都可以通过restfull api来有es通过9200端口进行交互

```xml
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```
VERB:  GET|POST|PUT|HEAD|DELETE

PROTOCOL:  https|http

PATH:  API 的终端路径（例如 `_count` 将返回集群中文档数量）。Path 可能包含多个组件，例如：`_cluster/stats` 和 `_nodes/stats/jvm`

QUERY\_STRING:  任意可选的查询字符串参数 (例如 `?pretty` 将格式化地输出 JSON 返回值，使其更容易阅读)

```xml
curl -XGET http://101.43.42.33:9200/_count?pretty #查询集群中文档数量
curl -i -XGET http://101.43.42.33:9200/_count?pretty #查询集群中文档数量,并返回请求头
```
#### es的面向文档
es的操作单位是文档（对象），并对文档（对象）中的每个内容做索引，使之可以被检索。

es选择JSON作为文档的序列化格式

#### 一个小案例
```xml
我们受雇于 Megacorp 公司，作为 HR 部门新的 “热爱无人机” （"We love our drones!"）激励项目的一部分，我们的任务是为此创建一个员工目录。该目录应当能培养员工认同感及支持实时、高效、动态协作，因此有一些业务需求：

支持包含多值标签、数值、以及全文本的数据
检索任一员工的完整信息
允许结构化搜索，比如查询 30 岁以上的员工
允许简单的全文搜索以及较复杂的短语搜索
支持在匹配文档内容中高亮显示搜索片段
支持基于数据创建和管理分析仪表盘
```
#### es中文档的增删改查
##### ·索引文档
es中的索引可以是一个动词，类似于插入。首先我们需要索引所有的员工文档，一个员工文档就是一个员工对象

```xml
索引（名词）：

如前所述，一个 索引 类似于传统关系数据库中的一个 数据库 ，是一个存储关系型文档的地方。 索引 (index) 的复数词为 indices 或 indexes 。

索引（动词）：

索引一个文档 就是存储一个文档到一个 索引 （名词）中以便被检索和查询。这非常类似于 SQL 语句中的 INSERT 关键词，除了文档已存在时，新文档会替换旧文档情况之外。

倒排索引：

关系型数据库通过增加一个 索引 比如一个 B树（B-tree）索引 到指定的列上，以便提升数据检索速度。Elasticsearch 和 Lucene 使用了一个叫做 倒排索引 的结构来达到相同的目的。

+ 默认的，一个文档中的每一个属性都是 被索引 的（有一个倒排索引）和可搜索的。一个没有倒排索引的属性是不能被搜索到的。我们将在 倒排索引 讨论倒排索引的更多细节。
```
```xml
PUT /megacorp/employee/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}

1: 表示该文档的id
employee：每个文档类型是employee类型的
megacorp：索引名称
```
##### ·检索文档
```xml
GET /megacorp/employee/1


返回结果
{
  "_index" :   "megacorp",
  "_type" :    "employee",
  "_id" :      "1",
  "_version" : 1,
  "found" :    true,
  "_source" :  {
      "first_name" :  "John",
      "last_name" :   "Smith",
      "age" :         25,
      "about" :       "I love to go rock climbing",
      "interests":  [ "sports", "music" ]
  }
}
```
##### ·更新文档
再次对某个id的文档执行put操作就是更新文档

```xml
PUT /megacorp/employee/2
{
    "first_name" :  "王",
    "last_name" :   "二麻子",
    "age" :         32,
    "about" :       "I like to collect rock albums",
    "interests":  [ "music" ]
}
```
##### ·删除文档
```xml
DELETE http://101.43.42.33:9200/megacorp/employee/6

返回结果
{
    "_index": "megacorp",
    "_type": "employee",
    "_id": "6",
    "_version": 2,
    "result": "deleted",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 5,
    "_primary_term": 1
}
```
##### ·检查文档是否存在
使用HEAD操作可以判断（未实验成功，直接检索指定id数据就可以查询索引是否包含该文档）

#### es中的各种复杂搜索
##### ·轻量搜索(\_search?q=key:value)
```xml
// 检索employee索引下的所有
GET http://101.43.42.33:9200/megacorp/employee/_search

GET http://101.43.42.33:9200/megacorp/employee/_search?q=last_name:志伟
```
##### ·表达式搜索
```xml
GET http://101.43.42.33:9200/megacorp/employee/_search
请求体(JSON)
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}

使用了match查询

```
##### ·复杂搜索
```xml
http://101.43.42.33:9200/megacorp/employee/_search

请求体
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "志伟" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 28 } 
                }
            }
        }
    }
}

年龄大于28 且last_name匹配志伟
```
##### ·全文搜索
```xml
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}

{
    "took": 0,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 1.4167401,
        "hits": [
            {
                "_index": "megacorp",
                "_type": "employee",
                "_id": "1",
                "_score": 1.4167401,
                "_source": {
                    "first_name": "贾",
                    "last_name": "志伟",
                    "age": 29,
                    "about": "I love to go rock climbing",
                    "interests": [
                        "sports",
                        "music"
                    ]
                }
            },
            {
                "_index": "megacorp",
                "_type": "employee",
                "_id": "2",
                "_score": 0.4589591,
                "_source": {
                    "first_name": "王",
                    "last_name": "二麻子",
                    "age": 32,
                    "about": "I like to collect rock albums",
                    "interests": [
                        "music"
                    ]
                }
            }
        ]
    }
}
表达式搜索发现得到两个结果，第二个结果并不匹配rock climbing但是还是返回了,因为about中有rock，所以也返回了，但是关联度没有那么高
```
##### ·短语检索
```xml
GET http://101.43.42.33:9200/megacorp/employee/_search

{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
当我们想精确搜索某些单词或者短语的时候就可以使用match_phrase
```
##### ·高亮搜索
```xml
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}

添加一个和query平级的参数highlight

{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.23013961,
      "hits": [
         {
            ...
            "_score":         0.23013961,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            },
            "highlight": {
               "about": [
                  "I love to go <em>rock</em> <em>climbing</em>" 
               ]
            }
         }
      ]
   }
}
```
##### ·分析
```xml
聚合统计所有员工的兴趣
GET /megacorp/employee/_search
{
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests" }
    }
  }
}

聚合还支持分级汇总 。比如，查询特定兴趣爱好员工的平均年龄：
GET /megacorp/employee/_search
{
    "aggs" : {
        "all_interests" : {
            "terms" : { "field" : "interests" },
            "aggs" : {
                "avg_age" : {
                    "avg" : { "field" : "age" }
                }
            }
        }
    }
}

聚合的组合使用
GET /megacorp/employee/_search
{
  "query": {
    "match": {
      "last_name": "smith"
    }
  },
  "aggs": {
    "all_interests": {
      "terms": {
        "field": "interests"
      }
    }
  }
}
```
·集群健康

```xml
GET http://101.43.42.33:9200/_cluster/health
```
