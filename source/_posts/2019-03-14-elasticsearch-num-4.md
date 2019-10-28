---
layout:     post
title:      "elastic入门(二)"
subtitle:   "elastic入门(二)"
date:       2019-03-14 14:30:00
author:     "Psl"
catalog:    true
tags:
  - elastic
---

## mapping

mapping是类似于数据库中的表结构定义，主要作用如下：

- 定义index下的字段名
- 定义字段类型，比如数值型、浮点型、布尔型等
- 定义倒排索引相关的设置，比如是否索引、记录position等

```http request
PUT test_index/_doc/1
{
  "username": "jack",
  "age": 15
}

GET test_index/_mapping
```

![](/img/in-post/2019-03-14/1.png)

### 自定义mapping

```http request
PUT my_index
{
  "mappings": {
    "dynamic": false,
      "properties": {
        "username": {
          "type": "keyword"
        },
        "age": {
          "type": "integer"
        }
      }
  }
}
```

mapping中的字段类型一旦设置，禁止直接修改，因为 lucene实现的倒排索引生成后不允许修改，应该重新建立新的索引，然后做reindex操作。

但是可以新增字段，通过 dynamic 参数来控制字段的新增，这个参数的值如下：

true：默认值，表示允许选自动新增字段
false：不允许自动新增字段，但是文档可以正常写入，但无法对字段进行查询等操作
strict：严格模式，文档不能写入，报错 

然后写入一个文档:

```http request
PUT my_index/_doc/1
{
  "username": "lili",
  "desc": "this is my index",
  "age": 20
}
```

>在mapping设置中，”dynamic”: false，表示在写入文档时，如果写入字段不存在也不会报错。这里的desc字段就是不存在的字段。

查询文档
```http request
GET my_index/_search
{
  "query": {
    "match": {
      "username": "lili"
    }
  }
}
```

```http request
GET my_index/_search
{
  "query": {
    "match": {
      "desc": "this is my index"
    }
  }
}
```

```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.2876821,
    "hits" : [
      {
        "_index" : "my_index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.2876821,
        "_source" : {
          "username" : "lili",
          "desc" : "this is my index",
          "age" : "20"
        }
      }
    ]
  }
}

```
```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}

```

对比两个结果可以看出能通过mapping中设置的字段查询到

### copy_to

作用是将该字段的值复制到目标字段，实现类似（6.0版本之前的）_all的作用。不会出现在_source中，只能用来搜索。

```http request
PUT my_index2
{
  "mappings": {
    "properties": {
      "first_name": {
        "type": "text",
        "copy_to": "full_name"
      },
      "last_name": {
        "type": "text",
        "copy_to": "full_name"
      },
      "full_name": {
        "type": "text"
      }
    }
  }
}
```

PUT：
```http request
PUT my_index2/_doc/1
{
  "first_name": "David",
  "last_name": "john"
}
```

查询：
```http request
GET my_index2/_search
{
  "query": {
    "match": {
      "full_name": {
        "query": "David john",
        "operator": "and"
      }
    }
  }
}
```

返回结果:

```json
{
  "took" : 316,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.5753642,
    "hits" : [
      {
        "_index" : "my_index2",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.5753642,
        "_source" : {
          "first_name" : "David",
          "last_name" : "john"
        }
      }
    ]
  }
}

```
>可以通过full_name来查询first_name，lastname两个字段，并且不区分大小写，但是一旦有一个字段的值匹配不上，就会返回为空

### Index

index参数作用是控制当前字段是否被索引，默认为true，false表示不记录，即不可被搜索。

```http request
PUT my_index3
{
  "mappings": {
    "properties": {
      "cookie": {
        "type": "text",
        "index": false
      },
      "content": {
        "type": "text",
        "index": true
      }
    }
  }
}
```

```http request
PUT my_index3/_doc/1
{
  "cookie": "efdfsdiadsasd",
  "content": "this is a cookie"
}
```
查询测试：
```http request
GET my_index3/_search
{
  "query": {
    "match": {
      "cookie": "efdfsdiadsasd"
    }
  }
}

GET my_index3/_search
{
  "query": {
    "match": {
      "content": "this is a cookie"
    }
  }
```
cookie字段不可被查询
```json
{
  "error": {
    "root_cause": [
      {
        "type": "query_shard_exception",
        "reason": "failed to create query: {\n  \"match\" : {\n    \"cookie\" : {\n      \"query\" : \"efdfsdiadsasd\",\n      \"operator\" : \"OR\",\n      \"prefix_length\" : 0,\n      \"max_expansions\" : 50,\n      \"fuzzy_transpositions\" : true,\n      \"lenient\" : false,\n      \"zero_terms_query\" : \"NONE\",\n      \"auto_generate_synonyms_phrase_query\" : true,\n      \"boost\" : 1.0\n    }\n  }\n}",
        "index_uuid": "5iLoJa4vRmiWA23sZ8-4TQ",
        "index": "my_index3"
      }
    ],
    "type": "search_phase_execution_exception",
    "reason": "all shards failed",
    "phase": "query",
    "grouped": true,
    "failed_shards": [
      {
        "shard": 0,
        "index": "my_index3",
        "node": "IWeH0fFvQD604ZiJ9OAdRw",
        "reason": {
          "type": "query_shard_exception",
          "reason": "failed to create query: {\n  \"match\" : {\n    \"cookie\" : {\n      \"query\" : \"efdfsdiadsasd\",\n      \"operator\" : \"OR\",\n      \"prefix_length\" : 0,\n      \"max_expansions\" : 50,\n      \"fuzzy_transpositions\" : true,\n      \"lenient\" : false,\n      \"zero_terms_query\" : \"NONE\",\n      \"auto_generate_synonyms_phrase_query\" : true,\n      \"boost\" : 1.0\n    }\n  }\n}",
          "index_uuid": "5iLoJa4vRmiWA23sZ8-4TQ",
          "index": "my_index3",
          "caused_by": {
            "type": "illegal_argument_exception",
            "reason": "Cannot search on field [cookie] since it is not indexed."
          }
        }
      }
    ]
  },
  "status": 400
}
```

### index_options

* index_options的作用是用于控制倒排索引记录的内容，有如下四种配置：
    - docs：只记录doc id
    - freqs：记录doc id 和term frequencies
    - positions：记录doc id、 term frequencies和term position
    - offsets：记录doc id、 term frequencies、term position、character offsets

* text类型的默认配置为positions，其他默认为docs。
* 记录的内容越多，占据的空间越大。

index_options设定：

```http request
PUT my_index4
{
  "mappings": {
    "properties": {
      "text": {
        "type": "text",
        "index_options": "offsets"
      }
    }
  }
}
```
PUT:
```http request
PUT my_index4/_doc/1
{
  "text": "Quick brown fox"
}
```

```http request
GET my_index4/_search
{
  "query": {
    "match": {
      "text": "brown fox"
    }
  },
  "highlight": {
    "fields": {
      "text": {} 
    }
  }
}
```

```json
{
  "took" : 68,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.5753642,
    "hits" : [
      {
        "_index" : "my_index4",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.5753642,
        "_source" : {
          "text" : "Quick brown fox"
        },
        "highlight" : {
          "text" : [
            "Quick <em>brown</em> <em>fox</em>"
          ]
        }
      }
    ]
  }
}
```
brown fox会被高亮显示

### null_value

这个参数的作用是当字段遇到null值的时候的处理策略，默认为null，即空值，此时es会忽略该值。可以通过这个参数设置某个字段的默认值

```http request
PUT my_index5
{
  "mappings": {
    "properties": {
      "status_code": {
        "type": "keyword",
        "null_value": "NULL"
      }
    }
  }
}

PUT my_index5/_doc/1
{
  "status_code": null
}

PUT my_index5/_doc/2
{
  "status_code": [] 
}

GET my_index5/_search
{
  "query": {
    "term": {
      "status_code": "NULL" 
    }
  }
}
```
```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.2876821,
    "hits" : [
      {
        "_index" : "my_index5",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.2876821,
        "_source" : {
          "status_code" : null
        }
      }
    ]
  }
}
```
1. 用术语null替换显式null值。
2. 空数组不包含显式null，因此不会用null_value替换。
3. 对NULL的查询返回文档1，而不是文档2。
>null_value需要与字段具有相同的数据类型。例如，长字段不能有字符串null_value。
null_value只影响数据的索引方式，它不修改_source文档。

更多详见[Mapping parameters](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-params.html)

### 数据类型

**核心数据类型**
- 字符串型：text、keyword（不会分词）
- 数值型：long、integer、short、byte、double、float、half_float等
- 日期类型：date
- 布尔类型：boolean
- 二进制类型：binary
- 范围类型：integer_range、float_range、long_range、double_range、date_range

**复杂数据类型**
- 数组类型：array
- 对象类型：object
- 嵌套类型：nested object
- 地理位置数据类型：geo_point、geo_shape
- 专用类型：ip（记录ip地址）、completion（实现自动补全）、token_count（记录分词数）、murmur3（记录字符串hash值）

**多字段特性**
- 多字段特性（multi-fields），表示允许对同一字段采用不同的配置，比如分词。

常见例子是对人名实现拼音搜索，只需要在人名中新增一个字段pinyin即可。但是这种方式不是十分优雅，multi-fields可以在不改变整体结构的前提下，增加一个子字段：

```http request
PUT my_index6
{
  "mappings": {
    "properties": {
      "username": {
        "type": "text",
        "fields": {
            "pinyin": {
                "type": "text",
                "analyzer": "pinyin"
            }
        }
      }
    }
  }
}
GET my_index6
{
  "query": {
    "match": {
      "username.pinyin": "pinyin"
    }
  }
}
```

### Dynamic mapping

Elasticsearch最重要的特性之一是，它试图摆脱您的阻碍，让您尽可能快地开始研究您的数据。要为文档建立索引，您不需要首先创建索引、定义映射类型和字段——您只需要为文档建立索引，索引、类型和字段就会自动出现:

默认情况下，当在文档中发现以前没有看到的字段时，Elasticsearch将把新字段添加到类型映射中。通过将动态参数设置为false(忽略新字段)或strict(遇到未知字段时抛出异常)，可以在文档和对象级别禁用此行为

字段映射规则

|JSON类型|ES类型|
|----|----|
| null |忽略 |
| true or false |boolean |
| floating point number | float|
| integer | long|
| object | object|
| array | 取决于数组中的第一个非空值。|
| string |要么是一个日期字段(如果值通过了日期检测)，要么是一个双字段或长字段(如果值通过了数值检测)，要么是一个带有关键字子字段的文本字段。|

```http request
DELETE test_index
PUT /test_index/_doc/1
{
  "username":"alfred",
  "age":1.2
}

GET test_index/_search
{
  "query": {
    "match": {
      "username.keyword": "alfred"
    }
  }
}
```

当你索引一个包含新字段的文档——一个之前没有的字段——Elasticsearch将使用动态映射猜测字段类型，这类型来自于JSON的基本数据类型，使用以下规则：

|JSON type                          |          Field type    |
|-----------------------------------|------------------------|
|Boolean: `true` or `false`         |          `"boolean"`   |
|Whole number: `123`                |          `"long"`      |
|Floating point: `123.45`           |          `"double"`    |
|String, valid date: `"2014-09-15"` |          `"date"`      |
|String: `"foo bar"`                |          `"string"`    |

> ### 注意
> 这意味着，如果你索引一个带引号的数字——`"123"`，它将被映射为`"string"`类型，而不是`"long"`类型。然而，如果字段已经被映射为`"long"`类型，Elasticsearch将尝试转换字符串为long，并在转换失败时会抛出异常。

### 日期自动识别

日期的自动识别可以自行配置日期的格式，默认情况下是：
```bash
["strict_date_opeional_time", "yyyy/MM/dd HH:mm:ss Z||yyyy/MM/dd Z"]
```
strict_date_opeional_time 是ISO 标准的日期格式，完整的格式如下：
```bash
YYYY-MM-DDhh:mm:ssTZD(eg:1997-07-16y19:20:30+01:00)
```
dynamic_date_formats：可以自定义日期类型 
date_detection：可以关闭日期自动识别机制（默认开启）
首先创建一个日期自动识别的索引：
```http request
DELETE my_index
PUT my_index
{
  "mappings": {
    "dynamic_date_formats": ["MM/dd/yyyy"]
  }
}

PUT my_index/_doc/1
{
  "create_time": "09/21/2016"
}

GET my_index/_mapping
```
```json
{
  "my_index" : {
    "mappings" : {
      "dynamic_date_formats" : [
        "MM/dd/yyyy"
      ],
      "properties" : {
        "create_time" : {
          "type" : "date",
          "format" : "MM/dd/yyyy"
        }
      }
    }
  }
}
```

关闭日期自动识别可以如下：

```http request
PUT my_index1
{
  "mappings": {
    "date_detection": false
  }
}

PUT my_index1/_doc/1 
{
  "create": "2015/09/02"
}
```

自定义时间类型:

```http request
PUT my_index2
{
  "mappings": {
    "dynamic_date_formats": ["MM/dd/yyyy"]
  }
}

PUT my_index2/_doc/1
{
  "create_date": "09/25/2015"
}
```

### 数字自动识别

字符串为数字的时候，默认不会自动识别为整型，因为字符串中出现数字是完全合理的。numeric_detection 可以开启字符串中数字的自动识别

```http request
DELETE my_index3
PUT my_index3
{
  "mappings": {
    "numeric_detection": true
  }
}

PUT my_index3/_doc/1
{
  "my_float":   "1.0", 
  "my_integer": "1" 
}

GET my_index3/_mapping
```

```json
{
  "my_index3" : {
    "mappings" : {
      "numeric_detection" : true,
      "properties" : {
        "my_float" : {
          "type" : "float"
        },
        "my_integer" : {
          "type" : "long"
        }
      }
    }
  }
}
```

### Dynamic Template

允许根据es自动识别的数据类型、字段名等动态设定字段类型，可以实现如下效果:
- 所有字符串都设定为`keyword`类型，即默认不分词
- 所有以message开头字段都设定为`text`类型，即分词
- 所有以long_开头的字段都设定为`long`类型
- 所有自动匹配为double类型的都设定为`float`类型，以节省空间

```http request
PUT test_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "keyword"
          }
        }
      }
    ]
  }
}

PUT test_index/_doc/1
{
  "name": "alfred"
}

GET test_index/_mapping 
```

匹配规则一般有如下几个参数：

- match_mapping_type: 匹配es自动识别的字段类型，如`boolean`,`long`,`string`等
- match,unmatch：匹配字段名
- path_match,path_unmatch: 匹配路径

**设置以message开头的字段都设置为text类型 （顺序由上而下）**
```http request
PUT test_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "message_as_text": {
          "match_mapping_type": "string",
          "match": "message*",
          "mapping": {
            "type": "text"
          }
        }
      },
      {
        "strings_as_keywords": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "keyword"
          }
        }
      }
    ]
  }
}
```
