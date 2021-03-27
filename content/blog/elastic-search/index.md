---
title: ES(ElasticSearch) 搜索基本概念简介
date: "2020-04-02T10:00Z"
description: 本篇文章介绍了 ES(ElasticSearch) 搜索相关的基本概念
---

### ES 的存储类型
ES 的存储类型大致分为两类
- 结构化数据 类型为 keyword、date、long、boolean 等
- 文本数据 类型为 text

存储索引建立规则：
- ES在存储结构化数据类型时，会将数据作为一个整体建立倒排索引。
- ES在存储文本数据类型时，会将文本先进行分词处理，然后按各个单独的词项建立倒排索引。

举例来说：

假设索引字段的定义如下：

字段名|字段类型|
---|---|
appName|keyword|
description|text|

当插入以下两条数据时

id|appName|description
---|---|---
1|App Store|App Store is Awesome
2|Apple Store| Store Sell Apple Pie

ES 会为 appName 字段建立如下的倒排索引：

索引 | 文档id 
---|---
App Store | 1
Apple Store |2

ES 会为 description 字段建立如下的倒排索引：

索引 | 文档id 
---|---
app | 1
store | 1, 2
awesome| 1
sell | 2
apple | 2
pie | 2

注：通用分词器会将词项转为小写

### ES 的搜索

ES 的搜索分为两大类
- 基于词项的搜索(Term 搜索)
- 基于全文的搜索

#### 基于词项的搜索
在进行基于词项的搜索，搜索词 与 文档字段需完全匹配。

ES 提供了五种基于词项的搜索方法
- term
- range
- exists
- prefix
- wildcard

##### DEMO 举例

id|appName|description
---|---|---
1|App Store|App Store is Awesome
2|Apple Store| Store Sell Apple Pie


```
{
    "query": {
        "term": {
            "appName": "App Store" // 可搜索到
            // "appName": "App"  // 无法搜索到
        }
    }
}
```

#### 基于全文的搜索

ES 主要提供了两种基于全文的搜索方法
- match
- match_phrase

在进行 match 搜索时，ES 会先将查询字符串进行分词，然后将每个词项与倒排索引进行匹配，任意一个词项匹配到，即搜索成功。

在进行 match_phrase 搜索时，ES 将查询字符串看作一个整体，只有文档中包含该短语字符串时，才搜索成功。

##### DEMO 

id|appName|description
---|---|---
1|App Store|App Store is Awesome
2|Apple Store| Store Sell Apple Pie

ES 会为 description 字段建立如下的倒排索引：

索引 | 文档id 
---|---
app | 1
store | 1, 2
awesome| 1
sell | 2
apple | 2
pie | 2

###### match 举例
```
{
    "query": {
        "match": {
            "description": "App" // 可以搜到文档1
          //"description": "App Store"  // 可以搜索到文档1，2 (会先将App Store分词, 通过 Store 搜到了文档2)
          //"description": "App Awesome"  // 可以搜索到文档1 (还是进行了分词)
        }
    }
}
```

##### match_phrase 举例
```
{
    "query": {
        "match_phrase": {
            "description": "App" // 搜到文档1
          //"description": "App Store"  // 搜索文档1 (App Store是一个整体)
          //"description": "App Awesome"  // 无法查到 (App Awesome是一个整体)
        }
    }
}
```

### 复合查询
以上为单条件查询，但通常业务上需要基于多个条件进行查询，ES 提供了 bool 复合查询，该查询可以包含4个查询字句，分别是
- must 必须全部符合该条件。贡献算分(算分可以简单理解为搜索的匹配度)
- should 符合条件之一即可。 贡献算分
- must_not 必须不符合该条件。 不贡献算分，有缓存优化
- filter 必须符合该条件。不贡献算分，有缓存优化

```
{
    "query": {
        "bool": {
            "must": [{  // 复合查询子句
                "term": {  // 基于词项搜索
                    "appName": "App Store"
                }
            }, {
                "range": { // 基于词项的范围搜索
                    "time": {
                        "lgt": "2019-01-01"
                    }
                }
            }, {
                "wildcard": { // 基于词项的通配符搜索
                    "appName": "* Store" 
                }
            }, {
                "match": { // 基于全文的 match 搜索
                    "description": "App Store"
                }
            }, {
                "match_phrase": { // 基于全文的 match_phrase 搜索
                    "description": "App Store"
                }
            }]
            "should": [{  // 复合查询子句
                ...
            }],
            "must_not": [{ // 复合查询子句
                ...
            }],
            "filter": [{ // 复合查询子句
                ...
            }]
        }
    }
}
```
一般情况下，在不需要算分的场景下，建议使用 filter 和 must_not, 因为不需要进行算分，ES 会有缓存优化。

另外每一个查询字句，可以继续嵌套 bool 查询。示意如下:
```
{
    "query": {
        "bool": {
            "must": [{  
                "bool": {  
                    "must":{...},
                    "should": {...},
                }
            }]
        }
    }
}
```

由于复合查询的 body 构造较为繁复，我们可以使用 [bodybuilder](https://github.com/danpaz/bodybuilder) 来简化这一过程。

示意如下：
```
var body = bodybuilder().query('match', 'message', 'this is a test').build()
// body == {
//   query: {
//     match: {
//       message: 'this is a test'
//     }
//   }
// }
```
详细[API文档](https://bodybuilder.js.org/docs/)