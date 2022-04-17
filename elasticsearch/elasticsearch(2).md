# ElasticSearch学习笔记(2)

主要是ES中的高级查询内容

## 说明

​	ES中提供了一种强大的检索数据方式,这种检索方式称之为`Query DSL`<Domain Specified Language> ,`Query DSL`是利用`Rest API传递JSON格式的请求体(Request Body)数据`与ES进行交互，这种方式的`丰富查询语法`让ES检索变得`更强大，更简洁`。

## 语法

```markdown
# GET /索引名/_doc/_search {json格式请求体数据}
```



**测试数据：**

```json
{
  "took" : 14,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "products",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "title" : "Iphone XR",
          "created_at" : "2018-09-01",
          "price" : "4999",
          "description" : "这是一部Iphone XR 手机"
        }
      },
      {
        "_index" : "products",
        "_type" : "_doc",
        "_id" : "brw9qX0BViY44oHwbDLo",
        "_score" : 1.0,
        "_source" : {
          "title" : "Iphone15",
          "created_at" : "2014-09-01",
          "price" : "4000",
          "description" : "这是一部过时了的Iphone 4手机"
        }
      },
      {
        "_index" : "products",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 1.0,
        "_source" : {
          "title" : "iphone14",
          "price" : 8999.99,
          "created_at" : "2021-09-15",
          "description" : "iPhone 14屏幕采用6.8英寸OLED屏幕"
        }
      },
      {
        "_index" : "products",
        "_type" : "_doc",
        "_id" : "4",
        "_score" : 1.0,
        "_source" : {
          "title" : "iphone15",
          "price" : 8999.99,
          "created_at" : "2021-10-15",
          "description" : "iPhone 15屏幕采用10.8英寸OLED屏幕"
        }
      }
    ]
  }
}

```

## 常见检索

### 查询所有[match_all]

> **match_all关键字:**  返回索引中的全部文档

```http
GET /products/_search
{
  "query": {
    "match_all": {}
  }
}
```

![image-20211211205159691](http://badwomen.asia/image-20211211205159691.png)

### 关键词查询[term]

这里引出关于分词器的问题了。后面会详细讲。

ES默认使用的是Standard Analyzer ，就是英文默认为一个单词为切割，中文默认为一个字为切割。

但是在Mappings映射中，只有text整个类型是可以被分词的，而keyword、date、integer等等类型其实是不支持分词的。

所以如果我们搜title的话，必须要整个搜索

```json
# 关键词查询 term
GET /products/_search
{
  "query": {
  	"term" : {
  		"title": {
 			 "value": "iphone"
      }
    }
  }
}

```

![image-20211211224820942](http://badwomen.asia/image-20211211224820942.png)

我们这样搜索是搜索不到的。

```json
# 关键词查询 term
GET /products/_search
{
  "query": {
    "term": {
      "description": {
        "value": "这"
      }
    }
  }
}
```

![image-20211211224920916](http://badwomen.asia/image-20211211224920916.png)

搜"这"是可以搜到的，但是搜“这里”就无法搜寻到。ES自带的分词器对于中文不友好。所以后期我们要更换分词器。

> 这里因为我的ES是7.16.0 版本，而常用的ES中文分词器IK分词器更新到7.15 所以暂时无法测试 我不想回退版本
>
> 等IK分词器更新在更新关于分词的一些问题

### 范围查询[range]

> **range 关键字**: 用来指定查询指定范围内的文档

```json
# 范围查询 range
GET /products/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 3000,
        "lte": 4999
      }
    }
  }
}
```

> lt 小于 gt 大于 e 等于

![image-20211211225222587](http://badwomen.asia/image-20211211225222587.png)



### 前缀查询 [prefix]

> **prefix 关键字**: 用来检索含有指定前缀的关键词的相关文档

```json
# 前缀查询 prefix
GET /products/_search 
{
  "query": {
    "prefix": {
      "title": {
        "value": "ipho"
      }
    }
  }
}
```



![image-20211211225528268](http://badwomen.asia/image-20211211225528268.png)



### 通配符查询[wildcard]

> **wildcard 关键字**: 通配符查询     **? 用来匹配一个任意字符  * 用来匹配多个任意字符**

```json
GET /products/_search
{
  "query": {
    "wildcard": {
      "description": {
        "value": "iphon*"
      }
    }
  }
}
```

### 多id查询[ids]

> **ids 关键字** : 值为数组类型,用来根据一组id获取多个对应的文档

![image-20211211230020402](http://badwomen.asia/image-20211211230020402.png)



### 模糊查询[fuzzy]

> **fuzzy 关键字**: 用来模糊查询含有指定关键字的文档

![image-20211211230213140](http://badwomen.asia/image-20211211230213140.png)

> 注意: `fuzzy 模糊查询  最大模糊错误 必须在0-2之间`
>
> - 搜索关键词长度为 2 不允许存在模糊
> - 搜索关键词长度为3-5 允许一次模糊
> - 搜索关键词长度大于5 允许最大2模糊

### 布尔查询[bool]

> **bool 关键字**: 用来组合多个条件实现复杂查询
>
> **must: 相当于&& 同时成立**
>
> **should: 相当于|| 成立一个就行**
>
> **must_not: 相当于!  不能满足任何一个**

![image-20211211230926965](http://badwomen.asia/image-20211211230926965.png)

### 多字段查询[multi_match]

```json
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "iphone13 毫",
      "fields": ["title","description"]
    }
  }
}
注意: 字段类型分词,将查询条件分词之后进行查询改字段  如果该字段不分词就会将查询条件作为整体进行查询
```

### 默认字段分词查询[query_string]

```json
GET /products/_search
{
  "query": {
    "query_string": {
      "default_field": "description",
      "query": "屏幕真的非常不错"
    }
  }
}
注意: 查询字段分词就将查询条件分词查询  查询字段不分词将查询条件不分词查询
```

![image-20211211231155420](http://badwomen.asia/image-20211211231155420.png)

### 高亮查询[highlight]

> **highlight 关键字**: 可以让符合条件的文档中的关键词高亮

```json
# 高亮查询
GET /products/_search 
{
  "query": {
    "multi_match": {
      "query": "iphone15",
      "fields": ["description","title"]
    }
  },
  "highlight": {
    "post_tags": ["</span>"], 
    "pre_tags": ["<span style='color:red'>"], 
    "fields": {
      "description": {},
      "title": {}
    }
  }
}
```

本质上是返回一个highlight的json字符串 然后加上标签

![image-20211211232511316](http://badwomen.asia/image-20211211232511316.png)



### 返回指定条数[size]

> 返回指定的条数 total: 4  但只返回了2条

![image-20211211232759366](http://badwomen.asia/image-20211211232759366.png)



### 分页查询[form]

```json
# 分页查询 
GET /products/_search
{
  "query": {
    "match_all": {}
  },
  "from": 0, # 跟mysql一样的分页 page * (size - 1)
  "size": 3
}
```



![image-20211211232947147](http://badwomen.asia/image-20211211232947147.png)

### 指定字段排序[sort]

```json
# 指定排序
GET /products/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "price": {
        "order": "asc"
      }
    }
  ]
}
```

> 跟 mysql 一样asc是升序，desc是降序

![image-20211211233126501](http://badwomen.asia/image-20211211233126501.png)



### 返回指定字段[_source]

```json
# 指定返回的字段
GET /products/_search
{
  "query": {
    "match_all": {}
  },
  "_source": ["title","description"]
}
```

![image-20211211233245661](http://badwomen.asia/image-20211211233245661.png)





## 过滤查询

### 过滤查询

过滤查询<filter query>，其实准确来说，ES中的查询操作分为2种: `查询(query)`和`过滤(filter)`。查询即是之前提到的`query查询`，它 (查询)默认会计算每个返回文档的得分，然后根据得分排序。而`过滤(filter)`只会筛选出符合的文档，并不计算 得分，而且它可以缓存文档 。所以，单从性能考虑，过滤比查询更快。 换句话说**`过滤适合在大范围筛选数据，而查询则适合精确匹配数据。一般应用时， 应先使用过滤操作过滤数据， 然后使用查询匹配数据。`** 

![image-20211101113527794](http://badwomen.asia/image-20211101113527794.png)

> 必须配合这布尔查询使用

![image-20211211233913962](http://badwomen.asia/image-20211211233913962.png)

### 类型

> 常见过滤类型有: term 、 terms 、ranage、exists、ids等filter。
