---
layout:     post
title:      "Elasticsearch Search API"
subtitle:   "Elasticsearch Search API"
date:       2019-03-15 14:30:00
author:     "Psl"
catalog:    true
tags:
  - elasticsearch
---

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/search.html)
实现对es中存储的数据进行查询分析，endpoint为`_search`，查询主要有两种形式：
- URI Search：操作简便，方便通过命令行测试，仅包含部分查询语法
- Request Body Search：es提供完备查询语法Query DSL

示例：
```http request
GET /user/_search?q=gender:M
GET /user/_search
{
  "query": {
    "match": {
      "gender": "M"
    }
  }
}
```

## URI Search详解
通过url query参数来实现搜索，常用参数如下：
- q: 指定查询语句，语法为 Query String Syntax
- df: q中不指定字段时默认查询的字段，如果不指定，es会查询所有字段
- sort：排序
- timeout：指定超时时间，默认不超时
- from,size：用于分页

```http request
GET /myindex/_search?q=alfred&df=user&sort=age:asc&from=4&size=10&timeout=1s
#查询user字段包含alfred的文档，结果按照age升序排列，返回第5-14个文档，如果超过1s没有结束，则以超时结束
```

## Query String Syntax

### 语法介绍
1. term与phrase
```markdown
alfred way 等效于 alfred OR way
"alfred way" 词语查询，要求先后顺序
```

2. 泛查询
```markdown
alfred等效于在所有字段去匹配该term
```

3. 指定字段
```markdown
name:alfred
```

4. Group分组指定，使用括号指定匹配的规则
```markdown
(quick OR brown) AND fox
status:(active OR pending) title:(full text search)
```
>q=<...>内容如果有空格会当作or处理

>加括号和不加括号的区别：加括号表示status是active或status是pengding结果，不加括号表示
status是active或者结果中包含pending

5. 布尔操作符
AND(&&),OR(||),NOT(!)
```markdown
name:(tom NOT lee) 注意大写，不能小写
```

6. "+ -"分别对应must和must_not
```markdown
name:(tom +lee -alfred)
等价于
name:((lee && !alfred)||(tom && lee && !alfred))
```
>+在url中会被解析为空格，要使用encode后的结果才可以，为%2B

7. 范围查询，支持数值和日志
- 区间写法，闭区间用[],开区间用{}
```markdown
age: [1 TO 10]意为 1<=age<=10
age: [1 TO 10}意为 1<=age<10
age: [1 TO ]意为 age>=1
age: [* TO 10]意为 age<=10
```
- 算数符号写法
```markdown
age:>=1
age:(>=1&&<=10)或者age:(+>=1 +<=10)
```

8. 通配符查询：？代表一个字符，*代表0或多个字符
```markdown
name:t?m
name:tom*

通配符匹配执行效率低，且占用较多内存，不建议使用
如无特殊需求，不要将?/*放在最前面
```

9. 正则表达式
```markdown
name:/[mb]oat/
```

10. 模糊匹配 fuzzy query
```markdown
name:roam~1
匹配与roam差一个character的词，比如foam roams等
```

11. 近似度查询 proximity search
```markdown
"fox quick"~5
允许fox 和quick 之间差5个词语
以term为单位进行差异比较，比如"quick fox" "quick brown fox"都会被匹配
```
### 查询示例

添加一些数据
```http request
DELETE test_search_index

PUT test_search_index
{
  "settings": {
    "index":{
        "number_of_shards": "1"
    }
  }
}

POST test_search_index/_bulk
{"index":{"_id":"1"}}
{"username":"alfred way","job":"java engineer","age":18,"birth":"1990-01-02","isMarried":false}
{"index":{"_id":"2"}}
{"username":"alfred","job":"java senior engineer and java specialist","age":28,"birth":"1980-05-07","isMarried":true}
{"index":{"_id":"3"}}
{"username":"lee","job":"java and ruby engineer","age":22,"birth":"1985-08-07","isMarried":false}
{"index":{"_id":"4"}}
{"username":"alfred junior way","job":"ruby engineer","age":23,"birth":"1989-08-07","isMarried":false}
```

查询所有关键词中有alfred的
```http request
GET test_search_index/_search?q=alfred
#profile设置为true可以分析elasticsearch的具体过程
GET test_search_index/_search?q=alfred
{
  "profile":true
}
```
查询所有关键词中有alfred的并指定为username字段
```http request
GET test_search_index/_search?q=username:alfred
```
查询所有关键词中有alfred或者way的并指定为username字段
```http request
GET test_search_index/_search?q=username:alfred way
```
如果加上""则表示关键词中必须包含alfred way
```http request
GET test_search_index/_search?q=username:"alfred way"
```

username字段中必须包含alfred和其它字段包含way的
```http request
GET test_search_index/_search?q=username:alfred AND way
```

username字段中必须包含alfred和way的
```http request
GET test_search_index/_search?q=username:(alfred AND way)
```

为了便于区别()的作用，在加入一条数据

```http request
POST test_search_index/doc/_bulk/
{"index":{"_id":"7"}}
{"username":"alfred","job":"java engineer way","age":18,"birth":"1990-01-02","isMarried":false}
```

```json
#q=username:alfred AND wayd的结果
{
  "took" : 3,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 3,
    "max_score" : 2.329832,
    "hits" : [
      {
        "_index" : "test_search_index",
        "_type" : "doc",
        "_id" : "7",
        "_score" : 2.329832,
        "_source" : {
          "username" : "alfred",
          "job" : "java engineer way",
          "age" : 18,
          "birth" : "1990-01-02",
          "isMarried" : false
        }
      },
      {
        "_index" : "test_search_index",
        "_type" : "doc",
        "_id" : "1",
        "_score" : 1.2048805,
        "_source" : {
          "username" : "alfred way",
          "job" : "java engineer",
          "age" : 18,
          "birth" : "1990-01-02",
          "isMarried" : false
        }
      },
      {
        "_index" : "test_search_index",
        "_type" : "doc",
        "_id" : "4",
        "_score" : 0.966926,
        "_source" : {
          "username" : "alfred junior way",
          "job" : "ruby engineer",
          "age" : 23,
          "birth" : "1989-08-07",
          "isMarried" : false
        }
      }
    ]
  }
}
```

```json
#q=username:(alfred AND way)的结果
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : 1.2048805,
    "hits" : [
      {
        "_index" : "test_search_index",
        "_type" : "doc",
        "_id" : "1",
        "_score" : 1.2048805,
        "_source" : {
          "username" : "alfred way",
          "job" : "java engineer",
          "age" : 18,
          "birth" : "1990-01-02",
          "isMarried" : false
        }
      },
      {
        "_index" : "test_search_index",
        "_type" : "doc",
        "_id" : "4",
        "_score" : 0.966926,
        "_source" : {
          "username" : "alfred junior way",
          "job" : "ruby engineer",
          "age" : 23,
          "birth" : "1989-08-07",
          "isMarried" : false
        }
      }
    ]
  }
}

```

username字段中必须包含alfred但是没有way的
```http request
GET test_search_index/_search?q=username:(alfred NOT way)
```
username必须含有way的
```http request
GET test_search_index/_search?q=username:(alfred +way)  +会被识别为空格
GET test_search_index/_search?q=username:(alfred %2Bway)
```

范围查询：
```http request
GET test_search_index/_search?q=username:alfred age:>26
#username字段包含alfred或者age大于26
GET test_search_index/_search?q=username:alfred AND age:>20
#username字段包含alfred并且age大于20
GET test_search_index/_search?q=birth:(>1980 AND <1990)
#birth字段在1980到1990之间
```
正则表达式和通配符:
```http request
GET test_search_index/_search?q=username:alf*
GET test_search_index/_search?q=username:/[a]?l.*/
```

模糊查询和近似度：
```http request
GET test_search_index/_search?q=username:alfed~1
GET test_search_index/_search?q=username:alfd~2
GET test_search_index/_search?q=job:"java engineer"
GET test_search_index/_search?q=job:"java engineer"~1
```

## Request Body Search

将查询语句通过http request body 发送到es，主要包含如下参数：
- query: 符合Query DSL语法的查询语句
- from,size
- timeout
- sort
- …

### Query DSL

Query DSL: 基于json定义的查询语言，主要包含如下两种类型:
- 字段类查询：如term、match、range等，只针对某一字段进行查询
- 复合查询：如bool查询等，包含一个或多个字段类查询或者复合查询语句

#### 字段类查询 
字段类查询主要包括以下两类：
- 全文匹配：针对text类型的字段进行全文检索，会对查询语句先进行分词处理，如match,match_phrase等query类型
- 单词匹配：不会对查询语句做分词处理，直接去匹配字段的倒排索引，如term,terms,range等query类型

查询示例

```http request
GET test_search_index/_search
{
  "query": {
    "match": {
      "username": "alfred way"
    }
  }
}
```
查询流程如下所示：
![](/img/in-post/2019-03-15/1.png)

通过operator参数可以控制单词间的匹配关系，可选项为or和and（默认为or） 
通过minimum_should_match参数可以控制需要匹配的单词数

```http request
GET test_search_index/_search
{
  "query": {
    "match": {
      "username": {
        "query": "alfred way", 
        "operator": "and"
      }
    }
  }
}

GET test_search_index/_search
{
  "query": {
    "match": {
      "username": {
        "query": "alfred way", 
        "minimum_should_match": 2
      }
    }
  }
}
#`alfred` 和 `way`必须同时存在才能满足
```

#### 相关性算分

相关性算分是指文档与查询语句间的相关度，英文为relevance 
通过倒排索引可以获取与查询语句相匹配的文档列表，那么如何将最符合用户查询需求的文档放到前列呢？本质是一个排序问题，排序依据是相关性算分。

相关性算分的几个重要概念：

- `Term Frequency（TF）`: 词频，即单词在该文档中出现的次数。词频越高，相关度越高
- `Document Frequency（DF）`: 文档频率，即单词出现的文档数
- `Inverse Document Frequency（IDF）`:逆向文档频率，与文档频率相反，简单理解为1/DF。即单词出现的文档数越少，相关度越高。
- `Field-length Norm`: 文档越短，相关性越高

ES目前主要有两个相关性算分模型：
- TF/IDF模型
- BM25模型：5.x之后的默认模型

可以通过explain参数来查看具体的计算方法，但要注意：es的算分是按照shard进行的，即shard分数计算时互相独立的，所以在使用explain的时候注意分片数；可以通过设置索引的分片数为1来避免这个问题。

```http request
{
  "explain": true,
  "query": {
    "match": {
      "FIELD": "TEXT"
    }
  }
}
```

#### Match-Phrase-Query

对字段作检索,有顺序要求，API示例：
```http request
GET test_search_index/_search
{
  "query": {
    "match_phrase": {
      "job": "engineer java"
    }
  }
}
```

通过slop参数可以控制单词间的间隔,类似url search里的近似度匹配

```http request
GET test_search_index/_search
{
  "query": {
    "match_phrase": {
      "job": {
        "query": "java engineer",
        "slop": 1
      }
    }
  }
}
```

#### Query-String-Query
类似于URI Search中的q参数查询

```http request
GET test_search_index/_search
{
  "profile": "true", 
  "query": {
    "query_string": {
      "default_field": "job",
      "query": "engineer AND java"
    }
  }
}
```
多字段查询
```http request
GET test_search_index/_search
{
  "profile": "true", 
  "query": {
    "query_string": {
      "fields": ["username", "job"],
      "query": "engineer AND alfred"
    }
  }
}
```
#### Simple-Query-String-Query
类似Query String ,但是会忽略错误的查询语法,并且仅支持部分查询语法
其常用的逻辑符号如下,不能使用AND, OR, NOT等关键词

- +代指AND
- |代指OR
- -代指NOT请求

```http request
GET test_search_index/_search
{
  "profile": "true", 
  "query": {
    "simple_query_string": {
      "fields": ["username"],
      "query": "alfred +way"
    }
  }
}
```

>与?q=alfred +way不同的是这里的alfred 和 way必须同时存在

#### Term-Terms-Query
将查询语句作为整个单词进行查询,即不对查询语句做分词处理:

```http request
GET test_search_index/_search
{
  "profile": "true", 
  "query": {
    "term": {
      "username": "alfred way"
    }
  }
}
```
一次传入多个单词进行查询:
```http request
GET test_search_index/_search
{
  "profile": "true", 
  "query": {
    "terms": {
      "username": [
        "alfred",
        "way"
      ]
    }
  }
}
```

#### Range Query
范围查询主要针对数值和日期类型:

```http request
GET test_search_index/_search
{
  "query": {
    "range": {
      "age": {
        "gt": 20,
        "lt": 40
      }
    }
  }
}
```

range 过滤器既能包含也能排除范围，通过下面的选项：
- gt: > 大于
- lt: < 小于
- gte: >= 大于或等于
- lte: <= 小于或等于

`range`过滤器也可以用于日期字段：
```json
"range":{
    "timestamp" : {
        "gt" : "2014-01-01 00:00:00",
        "lt" : "2014-01-07 00:00:00"
    }
}
```
这个过滤器将始终能找出所有时间戳大于当前时间减 1 小时的文档，让这个过滤器像移窗一样通过你的文档。

日期计算也能用于实际的日期，而不是仅仅是一个像 now 一样的占位符。只要在日期后加上双竖线 ||，就能使用日期数学表达式了。
```json
"range" : {
    "timestamp" : {
        "gt" : "2014-01-01 00:00:00",
        "lt" : "2014-01-01 00:00:00||+1M" <1>
    }
}
```
<1> 早于 2014 年 1 月 1 号加一个月
日期计算是与日历相关的，所以它知道每个月的天数，每年的天数，等等。更详细的关于日期的信息可以在这里找到
[日期格式手册](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/mapping-date-format.html)

`range`过滤器也可以用于字符串。字符串范围根据字典或字母顺序来计算。例如，这些值按照字典顺序排序：
假如我们想让范围从 a 开始而不包含 b，我们可以用类似的 range 过滤器语法：
```json
"range" : {
    "title" : {
        "gte" : "a",
        "lt" :  "b"
    }
}
```
>数字和日期字段的索引方式让他们在计算范围时十分高效。但对于字符串来说却不是这样。为了在字符串上执行范围操作，Elasticsearch 会在这个范围内的每个短语执行 term 操作。这比日期或数字的范围操作慢得多。
 字符串范围适用于一个基数较小的字段，一个唯一短语个数较少的字段。你的唯一短语数越多，搜索就越慢。
 
 #### 复合查询
 
复合查询是指包含字段类查询或复合查询的类型,主要包括以下几类:
- constant_score query
- bool query
- dis_max query
- function_score query
- boosting query 

**Constant Score Query**
该查询将其内部的查询结果文档得分都设定为1或者boost的值
多用于结合bool查询实现自定义得分
```http request
GET test_search_index/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "match": {
          "username": "alfred"
        }
      }
    }
  }
}
```
查询结果
```json
{
  "took" : 13,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "test_search_index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "username" : "alfred way",
          "job" : "java engineer",
          "age" : 18,
          "birth" : "1990-01-02",
          "isMarried" : false
        }
      },
      {
        "_index" : "test_search_index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "username" : "alfred",
          "job" : "java senior engineer and java specialist",
          "age" : 28,
          "birth" : "1980-05-07",
          "isMarried" : true
        }
      },
      {
        "_index" : "test_search_index",
        "_type" : "_doc",
        "_id" : "4",
        "_score" : 1.0,
        "_source" : {
          "username" : "alfred junior way",
          "job" : "ruby engineer",
          "age" : 23,
          "birth" : "1989-08-07",
          "isMarried" : false
        }
      }
    ]
  }
}
```
#### Bool Query
布尔查询由一个或多个布尔子句组成,主要包含如下4个:

|子句|含义|
|----|----|
|filter|只过滤符合条件的文档，不计算相关性算分|
|must|文档必须符合must中所有条件，影响相关性算分|
|must_not|文档必须不符合must中所有条件|
|should|文档可以符合should中的条件，不计算相关性算分|

**filter**
API:
```http request
GET test_search_index/_search
{
  "query": {
    "bool": {
      "must": [
        {}
      ],
      "must_not": [
        {}
      ],
      "should": [
        {}
      ],
      "filter": [
        {}
      ]
    }
  }
}
```

`Filter`查询只过滤符合条件的文档,不会进行相关性算分

es针对`filter`会有智能缓存,因此其执行效率很高
做简单匹配查询且不考虑算分时,推荐使用`filter`替代`query`等

```http request
GET test_search_index/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "username":"alfred"
          }
        }
      ]
    }
  }
}
```

**must**:
```http request
GET test_search_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "username":"alfred"
          }
        },
        {
          "match": {
            "job":"java"
          }
        }
      ]
    }
  }
}
```

返回结果:
```json
  {
    "_index" : "test_search_index",
    "_type" : "_doc",
    "_id" : "2",
    "_score" : 1.2314217,
    "_source" : {
      "username" : "alfred",
      "job" : "java senior engineer and java specialist",
      "age" : 28,
      "birth" : "1980-05-07",
      "isMarried" : true
    }
  },
  {
    "_index" : "test_search_index",
    "_type" : "_doc",
    "_id" : "1",
    "_score" : 1.1256535,
    "_source" : {
      "username" : "alfred way",
      "job" : "java engineer",
      "age" : 18,
      "birth" : "1990-01-02",
      "isMarried" : false
    }
  }
```

>`_score`为两个match查询到的分数之和

**must_not**:
```http request
GET test_search_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "username":"alfred"
          }
        }
      ],
      "must_not": [
        {
          "match": {
            "job":"specialist"
          }
        }
      ]
    }
  }
}
```

>匹配username包含`alfred`并且job不能包含`specialist`

**Should**:
- Should使用分两种情况:
    - bool查询中只包含should ,不包含must查询
    - bool查询中同时包含should和must查询
- 只包含should时,文档必须满足至少一个条件
    - minimum_should_match可以控制满足条件的个数或者百分比

```http request
GET test_search_index/_search
{
  "query": {
    "bool": {
      "should": [
        {"term": {"username": "alfred"}},
        {"term": {"username": "way"}},
        {"term": {"username": "junior"}}
      ], 
      "minimum_should_match": 2
    }
  }
}
```
```json
{
    "_index" : "test_search_index",
    "_type" : "_doc",
    "_id" : "4",
    "_score" : 1.8147054,
    "_source" : {
      "username" : "alfred junior way",
      "job" : "ruby engineer",
      "age" : 23,
      "birth" : "1989-08-07",
      "isMarried" : false
    }
  },
  {
    "_index" : "test_search_index",
    "_type" : "_doc",
    "_id" : "1",
    "_score" : 0.97797304,
    "_source" : {
      "username" : "alfred way",
      "job" : "java engineer",
      "age" : 18,
      "birth" : "1990-01-02",
      "isMarried" : false
    }
  }
```
>minimum_should_match加上后只返回了两个结果

同时包含should和must时,文档不必满足should中的条件,但是如果满足条件,会增加相关性得分

```http request
GET test_search_index/_search
{
  "query": {
    "bool": {
      "should": [
        {"term": {"job": "ruby"}}
      ],
      "must": [
        {"term": {"username": "alfred"}}
      ]
    }
  }
}
```
> `job` 为`ruby`的会增加分数排名更靠前

#### Query Context VS Filter Context
     
当一个查询语句位于Query或者Filter上下文时, es执行的结果会不同,对比如下:

|上下文类型|执行类型|使用方式|
|----|----|----|
|查找与查询语句最匹配的文档，对所有文档进行算分和排序|query<br>bool中的`must`和`should`|
|查找与查询语句相匹配的文档|bool中的`filter`与`must_not`<br>constant_score中的`filter`|

#### Count And Source Filtering
**count**

获取符合条件的文档数, endpoint为`_count`

**source filtering**
过滤返回结果中source中的字段,主要有如下几种方式:

**uri:**
```http request
GET test_search_index/_search?_source=username
```
**禁用source:**
```http request
GET test_search_index/_search
{
  "_source": false
}
```
**传字段名:**
```http request
GET test_search_index/_search
{
  "_source": ["username","job"]
}
```
**包括和不包括模糊匹配**
```http request
GET test_search_index/_search
{
  "_source": {
    "includes": "*i*",
    "excludes": "birth"
  }
}
```
