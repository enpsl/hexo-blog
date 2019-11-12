---
layout:     post
title:      "Elasticsearch Search运行机制"
subtitle:   "Elasticsearch Search运行机制"
date:       2019-11-12 14:30:00
author:     "Psl"
catalog:    true
tags:
  - elasticsearch
---

## Query-Then-Fetch
Search执行的时候实际分两个步骤运作的
- Query阶段
- Fetch阶段

在es里被称为Query-Then-Fetch机制

### Search的运行机制-Query阶段
node3在接收到用户的search请求后,会先进行Query阶段(此时是CoordinatingNode角色)
node3在6个主副分片中随机选择3个分片,发送search request
被选中的3个分片会分别执行查询并排序,返回from+size个文档Id和排序值
node3整合3个分片返回的from+size个文档Id ,根据排序值排序后选取from到from+size的文档Id

```http request
GET movies/_search
{
  "from": 10,
  "size": 20
}
```
>表示从第10个文档开始，截取第10-29的文档

### Search的运行机制-Fetch阶段

node3根据Query阶段获取的文档Id列表去对应的shard上获取文档详情数据a
- node3向相关的分片发送multi-get请求
- 3个分片返回文档详细数据
- node3拼接返回的结果并返回给客户

### Search的运行机制-相关性算分问题
相关性算分在shard与shard间是相互独立的,也就意味着同一个Term的IDF等值在不同shard上是不同的。文档的相关性算分和它所处的shard相关

在文档数量不多时,会导致相关性算分严重不准的情况发生

加上`explain`参数可查看文档内容所在shard
```http request
GET movies/_search
{
  "explain": true, 
  "query": {
    "match": {
      "genre": {
        "query": "Comedy Children",
        "operator": "and"
      }
    }
  }
}
```

解决思路有两个:

一是设置分片数为1个,从根本上排除问题,在文档数量不多的时候可以考虑该方。案,比如百万到千万级别的文档数量
二是使用`DFS Query-then-Fetch`查询方式.`DFS Query-then-Fetch`是在拿到所有文档后再重新完整的计算一次相关性算分,耗费更多的cpu和内存,执行性能也比较低下,一般不建议使用。使用方式如下:

```http request
GET movies/_search?search_type=dfs_query_then_fetch
{
  "explain": false, 
  "query": {
    "match": {
      "genre": {
        "query": "Comedy Children",
        "operator": "and"
      }
    }
  }
}
```

### sorting

es默认会采用相关性算分排序,用户可以通过设定`sort`参数来自行设定排序规则

排序的过程实质是对字段原始内容排序的过程,这个过程中倒排索引无法发挥作用,需要用到正排索引,也就是通过文档Id和字段可以快速得到字段原始内容。

es对此提供了2种实现方式:

- fielddata默认禁用
- doc values默认启用,除了text类型

```http request
GET movies/_search
{
  "sort": [
    {
      "title": {
        "order": "desc"
      }
    }
  ]
}
```
response:

```json
{
  "error": {
    "root_cause": [
      {
        "type": "illegal_argument_exception",
        "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [title] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory. Alternatively use a keyword field instead."
      }
    ],
    "type": "search_phase_execution_exception",
    "reason": "all shards failed",
    "phase": "query",
    "grouped": true,
    "failed_shards": [
      {
        "shard": 0,
        "index": "movies",
        "node": "6pljLZ0vQQKObirnkOOsnw",
        "reason": {
          "type": "illegal_argument_exception",
          "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [title] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory. Alternatively use a keyword field instead."
        }
      }
    ],
    "caused_by": {
      "type": "illegal_argument_exception",
      "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [title] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory. Alternatively use a keyword field instead.",
      "caused_by": {
        "type": "illegal_argument_exception",
        "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [title] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory. Alternatively use a keyword field instead."
      }
    }
  },
  "status": 400
}
```
解决办法:
- 使用keyword排序
- fielddata开启
```http request
GET movies/_search
{
  "sort": [
    {
      "title.keyword": {
        "order": "desc"
      }
    }
  ]
}
```

### Fileddata vs DocValues

|对比|Fileddata|DocValues|
|----|----|----|
|创建时机|搜索时即时创建|索引时创建|
|创建位置|JVM Heap|磁盘|
|优点|不占用磁盘空间|不占用Heap内存|
|缺点|文档过多时，即时创建花费时间长，内存占用高|减慢索引的速度，占用磁盘空间|

Fielddata默认是关闭的,可以通过如下api开启:
- 此时字符串是按照分词后的term排序,往往结果很难符合预期
- 一般是在对分词做聚合分析的时候开启,

```http request
PUT movies/_mapping
{
  "properties": {
    "title": {
      "type": "text",
      "fielddata": true     # 可随时开启
    }
  }
}
# 开启后在查询不会报错
GET movies/_search
{
  "sort": [
    {
      "title.keyword": {
        "order": "desc"
      }
    }
  ]
}
```

Doc Values默认是启用的,可以在创建索引的时候关闭:-如果后面要再开启doc values ,需要做reindex操作

```http request
DELETE test_doc_values
PUT test_doc_values
PUT test_doc_values/_mapping
{
  "properties": {
    "username": {
      "type": "keyword",
      "doc_values": false
    }
  }
}
```

关闭后在排序会报错

```http request
GET test_doc_values/_search
{
  "sort": [
    {
      "username": {
        "order": "desc"
      }
    }
  ]
}
```

```json
{
  "error": {
    "root_cause": [
      {
        "type": "illegal_argument_exception",
        "reason": "Can't load fielddata on [username] because fielddata is unsupported on fields of type [keyword]. Use doc values instead."
      }
    ],
    "type": "search_phase_execution_exception",
    "reason": "all shards failed",
    "phase": "query",
    "grouped": true,
    "failed_shards": [
      {
        "shard": 0,
        "index": "test_doc_values",
        "node": "6pljLZ0vQQKObirnkOOsnw",
        "reason": {
          "type": "illegal_argument_exception",
          "reason": "Can't load fielddata on [username] because fielddata is unsupported on fields of type [keyword]. Use doc values instead."
        }
      }
    ],
    "caused_by": {
      "type": "illegal_argument_exception",
      "reason": "Can't load fielddata on [username] because fielddata is unsupported on fields of type [keyword]. Use doc values instead.",
      "caused_by": {
        "type": "illegal_argument_exception",
        "reason": "Can't load fielddata on [username] because fielddata is unsupported on fields of type [keyword]. Use doc values instead."
      }
    }
  },
  "status": 400
}
```

>text类型不支持Doc Values,Fielddata只能支持与text类型

### 分页与遍历
分页与遍历-fromsize

es提供了3种方式来解决分页与遍历的问题:
- from/size
- scroll
- search_after
from/size最常用的分页方案

from指明开始位置size指明获取总数

**深度分页是一个经典的问题:在数据分片存储的情况下如何获取前1000个文档?**
获取从990-1000的文档时,会在每个分片上都先获取1000个文档,然后再由.Coordinating Node聚合所有分片的结果后再排序选取前1000个文档
页数越深,处理文档越多,占用内存越多,耗时越长,尽量避免深度分页, es通过 index.max_result_window限定最多到10000条数据

```http request
GET movies/_search
{
  "from": 10,
  "size": 10000
}

GET movies/_search
{
  "from": 10000,
  "size": 10
}
```
response
```json
{
  "error": {
    "root_cause": [
      {
        "type": "illegal_argument_exception",
        "reason": "Result window is too large, from + size must be less than or equal to: [10000] but was [10010]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level setting."
      }
    ],
    "type": "search_phase_execution_exception",
    "reason": "all shards failed",
    "phase": "query",
    "grouped": true,
    "failed_shards": [
      {
        "shard": 0,
        "index": "movies",
        "node": "6pljLZ0vQQKObirnkOOsnw",
        "reason": {
          "type": "illegal_argument_exception",
          "reason": "Result window is too large, from + size must be less than or equal to: [10000] but was [10010]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level setting."
        }
      }
    ],
    "caused_by": {
      "type": "illegal_argument_exception",
      "reason": "Result window is too large, from + size must be less than or equal to: [10000] but was [10010]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level setting.",
      "caused_by": {
        "type": "illegal_argument_exception",
        "reason": "Result window is too large, from + size must be less than or equal to: [10000] but was [10010]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level setting."
      }
    }
  },
  "status": 400
}
```

### 分页与遍历-scroll
遍历文档集的api ,以快照的方式来避免深度分页的问题

- 不能用来做实时搜索,因为数据不是实时的
- 尽量不要使用复杂的sort条件,使用_doc最高效
- 使用稍嫌复杂

第一步需要发起1个scroll search:

es在收到该请求后会根据查询条件创建文档Id合集的快照

```http request
GET movies/_search?scroll=5m
{
  "size": 1
}
5m至快照有效时常，size返回条数
```
response
```json
{
  "_scroll_id" : "DnF1ZXJ5VGhlbkZldGNoAgAAAAAAAAagFnY3WnRoVnRLVDNtWncyZ3k4QzlKU1EAAAAAAAAGnxZ2N1p0aFZ0S1QzbVp3Mmd5OEM5SlNR",
  "took" : 3,
  "timed_out" : false,
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 9743,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "movies",
        "_type" : "_doc",
        "_id" : "movieId",
        "_score" : 1.0,
        "_source" : {
          "year" : 0,
          "title" : "title",
          "@version" : "1",
          "genre" : [
            "genres"
          ],
          "id" : "movieId"
        }
      }
    ]
  }
}

```
第二步调用scroll search的api ,获取文档集合

不断迭代调用直到返回hits.hits数组为空时停止
```http request
POST _search/scroll
{
  "scroll": "5m",
  "scroll_id": "DnF1ZXJ5VGhlbkZldGNoAgAAAAAAAAagFnY3WnRoVnRLVDNtWncyZ3k4QzlKU1EAAAAAAAAGnxZ2N1p0aFZ0S1QzbVp3Mmd5OEM5SlNR"
}
```
过多的scroll调用会占用大量内存,可以通过clear api删除过多的scroll快照:
```http request
DELETE _search/scroll
{
  "scroll_id":["DnF1ZXJ5VGhlbkZldGNoAgAAAAAAAAagFnY3WnRoVnRLVDNtWncyZ3k4QzlKU1EAAAAAAAAGnxZ2N1p0aFZ0S1QzbVp3Mmd5OEM5SlNR"]
}

DELETE _search/scroll/_all
```

### 分页与遍历-search_after

避免深度分页的性能问题,提供实时的下一页文档获取功能

- 缺点是不能使用from参数,即不能指定页数
- 只能下一页,不能上一页
- 使用简单

第一步为正常的搜索,但要指定sort值,并保证值唯一
第二步为使用上一步最后一个文档的sort值进行查询

```http request
GET movies/_search
{
  "size": 1, 
  "sort": [
    {
      "year": "desc",
      "title": "desc"
    }
  ]
}
```

```json
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 9743,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "movies",
        "_type" : "_doc",
        "_id" : "187717",
        "_score" : null,
        "_source" : {
          "year" : 2018,
          "title" : "Won't You Be My Neighbor?",
          "@version" : "1",
          "genre" : [
            "Documentary"
          ],
          "id" : "187717"
        },
        "sort" : [
          2018,
          "you"
        ]
      }
    ]
  }
}

```

```http request
GET movies/_search
{
  "size": 1,
  "sort": [
    {
      "year": "desc",
      "title": "desc"
    }
  ],
  "search_after": [2018,"you"]
}
```

如何避免深度分页问题?
通过唯一排序值定位将每次要处理的文档数都控制在size内

返回排序值在search after之后的size个文档，假如size=10,有5个分片，Coordinating Node只需要聚合5个分片取10个文档共50个文档，然后在取前10个文档返回即可
有效的控制了聚合的文档的数量

### 场景:

|类型|场景|
|----|----|
|from/size|需要实时获取顶部的文档，可自由分页|
|scroll|需要全部文档，常用于导出，数据非实时|
|search after|需要全部的文档，不需要自由分页，数据实时|

[search 官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/search.html)