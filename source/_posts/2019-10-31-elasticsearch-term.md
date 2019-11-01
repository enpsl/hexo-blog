---
layout:     post
title:      "Elasticsearch Term Match"
subtitle:   "Elasticsearch Term Terms Match MatchAll"
date:       2019-10-31 14:30:00
author:     "Psl"
catalog:    true
tags:
  - elasticsearch
---

## Term
`term query`会去倒排索引中寻找确切的`term`，它并不知道分词器的存在，这种查询适合`keyword`、`numeric`、`date`等明确值的

term：查询某个字段里含有某个关键词的文档

```http request
GET test_search_index/_search
{
  "query": {
    "term": {
      "username": "alfred"
    }
  }
}
```
返回结果
```json
"hits" : [
      {
        "_index" : "test_search_index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 0.636667,
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
        "_score" : 0.48898652,
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
        "_id" : "4",
        "_score" : 0.39691794,
        "_source" : {
          "username" : "alfred junior way",
          "job" : "ruby engineer",
          "age" : 23,
          "birth" : "1989-08-07",
          "isMarried" : false
        }
      }
    ]
```
发现，`username`里有关`alfred`的关键字都查出来了，但是我只想精确匹配`alfred way`这个，按照下面的写法看看能不能查出来：

```http request
GET test_search_index/_search
{
  "query": {
    "term": {
      "username": "alfred way"
    }
  }
}
```

执行发现无数据，从概念上看，`term`属于精确匹配，只能查单个词。我想用`term`匹配多个词怎么做？可以使用terms来：

```http request
GET test_search_index/_search
{
  "query": {
    "terms": {
      "username": ["alfred", "way"]
    }
  }
}
```

发现全部查询出来，为什么？因为`terms`里的`[ ]`多个是或者的关系，只要满足其中一个词就可以。想要通知满足两个词的话，就得使用在search api那篇中提到的`bool`查询来做了

match查询：

match query 知道分词器的存在，会对field进行分词操作，然后再查询
```http request
GET test_search_index/_search
{
  "query": {
    "match": {
      "username": "alfred"
    }
  }
}
```
match_all：查询所有文档
{ "match_all": {}} 匹配所有的， 当不给查询条件时，默认。
```http request
GET test_search_index/_search
{
  "query": {
    "match_all": {
      "boost": 1.2
    }
  }
}
```
`_score`随boost值改变而改变:

multi_match：可以指定多个字段
```http request
GET test_search_index/_search
{
  "profile": "true", 
  "query": {
    "multi_match": {
      "query" : "alfred java",
      "fields":  ["username","job"]
    }
  }
}
```
返回结果：
```json
"hits" : [
  {
    "_index" : "test_search_index",
    "_type" : "_doc",
    "_id" : "1",
    "_score" : 0.636667,
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
    "_score" : 0.636667,
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
    "_id" : "3",
    "_score" : 0.48898652,
    "_source" : {
      "username" : "lee",
      "job" : "java and ruby engineer",
      "age" : 22,
      "birth" : "1985-08-07",
      "isMarried" : false
    }
  },
  {
    "_index" : "test_search_index",
    "_type" : "_doc",
    "_id" : "4",
    "_score" : 0.39691794,
    "_source" : {
      "username" : "alfred junior way",
      "job" : "ruby engineer",
      "age" : 23,
      "birth" : "1989-08-07",
      "isMarried" : false
    }
  }
]
```
