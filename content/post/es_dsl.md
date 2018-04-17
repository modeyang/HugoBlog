---
title: "Elasticsearch-dsl查询"
date: 2018-04-16T18:52:26+08:00
categories:
- ELK
tags:
- elasticsearch
- query
comments: true
---
- 这个才是实际最常用的方式，可以构建复杂的查询条件。
- 不用一开始就想着怎样用 Java Client 端去调用 Elasticsearch 接口。DSL 会了，Client 的也只是用法问题而已。

<!--more-->

# 1. Term query
用于精确查找在inverted index里面，如下面：
```bash
GET /_search
{
  "query": {
    "bool": {
      "should": [
        {
          "term": {
            "status": {
              "value": "urgent",
              "boost": 2.0 
            }
          }
        },
        {
          "term": {
            "status": "normal" 
          }
        }
      ]
    }
  }
}
```
## 1.1 原理
String字段可以是analyzed（用于全文搜索）或者not_analyzed（用于精确查找）Exact values（如数字，日期或者not_analyzed字符串）放在inverted index中是他的本身值。

默认的string字段是可分析的，因此string值首先会被analyzer分解成一系列的term, 然后加入到inverted index中。

默认çanalyzer的工作原理： 
1. drops most punctuation  
2. breaks up text into individual words, and lower cases them

例如： **“Quick Brown Fox!”** into the terms **[quick, brown, fox]**.

所以对于term查询来说，是在inverted index中去å¯»找精确的term value，而不必去考虑字段的analyze情况，这个match查询正好相反。

# 2. Terms Query
```json
{
    "constant_score" : {
        "filter" : {
            "terms" : { "user" : ["kimchy", "elasticsearch"]}
        }
    }
}
```
terms查询也叫作in， 当用于简单的filter时。
## 2.1 terms 查询机制
现在还不知道具体怎么用。
A concrete example would be to filter tweets tweeted by your followers。
```bash
# index the information for user with id 2, specifically, its followers
curl -XPUT localhost:9200/users/user/2 -d '{
   "followers" : ["1", "3"]
}'

# index a tweet, from user with id 1
curl -XPUT localhost:9200/tweets/tweet/1 -d '{
   "user" : "1"
}'

# search on all the tweets that match the followers of user 2
curl -XGET localhost:9200/tweets/_search -d '{
  "query" : {
    "terms" : {
      "user" : {
        "index" : "users",
        "type" : "user",
        "id" : "2",
        "path" : "followers"
      }
    }
  }
}'
```
# 3. Range Query
基于区间查询，根据字段类型来算，string类型属于TermRangeQuery, 而数字与时间类型为NumericRangeQuery.
```bash
{
    "range" : {
        "age" : {
            "gte" : 10,
            "lte" : 20,
            "boost" : 2.0
        }
    }
}

## 基于时间
{
    "range" : {
        "date" : {
            "gte" : "now-1d/d",
            "lt" :  "now/d"
        }
    }
}

## 自定义时间格式区间查询
{
    "range" : {
        "born" : {
            "gte": "01/01/2012",
            "lte": "2013",
            "format": "dd/MM/yyyy||yyyy"
        }
    }
}
```
时间区间内容[算法](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#date-math).

# 4. Exists Query
返回的documents中 field至少有一ä¸ªnon-null的值
```bash
{
    "exists" : { "field" : "user" }
}

# missing 查询
"bool": {
    "must_not": {
        "exists": {
            "field": "user"
        }
    }
}
```
# 5. Wildcard Query
```bash
# 使用* ？做简单的字符替换查询
{
    "wildcard" : { "user" : { "value" : "ki*y", "boost" : 2.0 } }
}
```
# 6. Regexp Query
```bash
{
    "regexp":{
        "name.first": "s.*y"
    }
}
```
# 7. [Fuzzy Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-fuzzy-query.html) 
有点类似基于** Levenshtein edit distance**算法， +/- margin on numeric and date fields.
```bash
{
    "fuzzy" : {
        "user" : {
            "value" :         "ki",
            "boost" :         1.0,
            "fuzziness" :     2,
            "prefix_length" : 0,
            "max_expansions": 100
        }
    }
}

# 基于数字区间  10 - 14之间
{
    "fuzzy" : {
        "price" : {
            "value" : 12,
            "fuzziness" : 2
        }
    }
}

# 基于时间区间
{
    "fuzzy" : {
        "created" : {
            "value" : "2010-02-05T12:05:07",
            "fuzziness" : "1d"
        }
    }
}
```

# 8. Type Query
```json
{
    "type" : {
        "value" : "my_type"
    }
}
```

# 9. Ids Query
```json
{
    "ids" : {
        "type" : "my_type",
        "values" : ["1", "4", "100"]
    }
}
```

