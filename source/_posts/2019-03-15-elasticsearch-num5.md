---
layout:     post
title:      "Elasticsearch Search API"
subtitle:   "Elasticsearch Search API"
date:       2019-03-15 14:30:00
author:     "Psl"
catalog:    true
tags:
  - elastic
---

[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/search.html)
实现对es中存储的数据进行查询分析，endpoint为_search，查询主要有两种形式：
- URI Search：操作简便，方便通过命令行测试，仅包含部分查询语法
- Request Body Search：es提供完备查询语法Query DSL

示例：
```http request
GET /user/_search?q=gender:M
GET /user/profile/_search
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

POST test_search_index/doc/_bulk
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
#GET test_search_index/_search?q=username:(alfred +way)  +会被识别为空格
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
```

#### 相关性算分

相关性算分是指文档与查询语句间的相关度，英文为relevance 
通过倒排索引可以获取与查询语句相匹配的文档列表，那么如何将最符合用户查询需求的文档放到前列呢？本质是一个排序问题，排序依据是相关性算分。

相关性算分的几个重要概念：

- Term Frequency（TF）: 词频，即单词在该文档中出现的次数。词频越高，相关度越高
- Document Frequency（DF）: 文档频率，即单词出现的文档数
- Inverse Document Frequency（IDF）:逆向文档频率，与文档频率相反，简单理解为1/DF。即单词出现的文档数越少，相关度越高。
- Field-length Norm: 文档越短，相关性越高

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