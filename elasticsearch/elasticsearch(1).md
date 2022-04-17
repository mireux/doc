# ES学习笔记（1）

ES和Kibana下载就不详细说了，网上都有，这里主要介绍ES的使用和关于SpringBoot的整合。

## 核心概念

### 索引<Index> 

**`一个索引就是一个拥有几分相似特征的文档的集合`**。比如说，你可以有一个商品数据的索引，一个订单数据的索引，还有一个用户数据的索引。**`一个索引由一个名字来标识`**`(必须全部是小写字母的)`**，**并且当我们要对这个索引中的文档进行索引、搜索、更新和删除的时候，都要使用到这个名字。

### 映射<Mapping>

**`映射是定义一个文档和它所包含的字段如何被存储和索引的过程`**。在默认配置下，ES可以根据插入的数据`自动地创建mapping，也可以手动创建mapping`。 mapping中主要包括字段名、字段类型等 

### 文档<Document>

**`文档是索引中存储的一条条数据。一条文档是一个可被索引的最小单元`**。ES中的文档采用了轻量级的JSON格式数据来表示。 





## 基本操作

### 索引

#### 查询索引

```markdown
# 查询索引
- GET /products/_cat/indices?v
```

![image-20211211144333126](http://badwomen.asia/image-20211211144333126.png)

 #### 创建索引

```markdown
# 1.创建索引
- PUT /索引名 ====> PUT /products
- 注意: 
		1.ES中索引健康转态  red(索引不可用) 、yellwo(索引可用,存在风险)、green(健康)
		2.默认ES在创建索引时回为索引创建1个备份索引和一个primary索引
```

![image-20211211144409292](http://badwomen.asia/image-20211211144409292.png)

> 如果不手动设置 那么索引为黄色 存在风险

![image-20211211144502384](http://badwomen.asia/image-20211211144502384.png)

```markdown
		
# 2.创建索引 进行索引分片配置
- PUT /products
{
  "settings": {
    "number_of_shards": 1, #指定主分片的数量
    "number_of_replicas": 0 #指定副本分片的数量
  }
}
```



![image-20211211144644962](http://badwomen.asia/image-20211211144644962.png)

![image-20211211144649950](http://badwomen.asia/image-20211211144649950.png)



#### 删除索引

最简单

```markdown
# 删除索引
- DELETE /products
```





### 映射<Mapping>

#### 创建

**映射的类型：**

字符串类型: keyword 关键字 关键词 、text 一段文本

数字类型：integer long  

小数类型：float double

布尔类型：boolean 

日期类型：date

```markdown
# 创建索引&映射
```

索引和映射可以同时创建

我们创建一个商品映射作为后面的例子

其中包括 title 、 price 、 created_at 、 description

```json
# 创建索引的同时创建映射
PUT /products
{
  "settings": {
    "number_of_replicas": 0, 
    "number_of_shards": 1
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "keyword"
      },
      "price": {
        "type": "float"
      },
      "created_at": {
        "type": "date"
      },
      "description": {
        "type": "text"
      }
    }
  }
}
```



![image-20211211145206537](http://badwomen.asia/image-20211211145206537.png)

#### 查询

```markdown
# 查询某个索引的映射
- GET /索引名/_mapping =====> GET /products/_mapping
```

![image-20211211145345896](http://badwomen.asia/image-20211211145345896.png)

### 文档<document>

#### 添加文档

```http
# 添加文档
POST /products/_doc/1 #指定id生成文档
{
  "title": "Iphone XR",
  "created_at": "2018-09-01",
  "price": "4999",
  "description": "这是一部Iphone XR 手机"
}

```

> 指定id添加

![image-20211211191401607](http://badwomen.asia/image-20211211191401607.png)



> 不指定id 添加

![image-20211211191742870](http://badwomen.asia/image-20211211191742870.png)

#### 查询文档

```http
# 查询文档
GET /products/_doc/1
```

![image-20211211191523322](http://badwomen.asia/image-20211211191523322.png)



#### 删除文档

```json
# 删除文档
DELETE /products/_doc/bLw1qX0BViY44oHwqzJq
```

```json
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "bLw1qX0BViY44oHwqzJq",
  "_version" : 2,
  "result" : "deleted",
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 2,
  "_primary_term" : 1
}

```

#### 更新文档

更新文档有两种方式：

- 一种是PUT /索引名/_doc/id 这种方式是先删除文档内容，再插入更新内容
- 一种是POST /索引名/_doc/id/ _update 这种是在原先的文档内容上更新





> 我们先插入一条数据

```json
POST /products/_doc
{
  "title":"Iphone 4",
  "created_at": "2014-09-01",
  "price": "499",
  "description": "这是一部过时了的Iphone 4手机"
}



{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "bbw6qX0BViY44oHwSzKn",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 3,
  "_primary_term" : 1
}

```



> 第一种方式更新：

```json
# 第一种方式更新 不保存原文档的内容
PUT /products/_doc/bbw6qX0BViY44oHwSzKn
{
  "title": "Iphone15"
}


{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "bbw6qX0BViY44oHwSzKn",
  "_version" : 2,
  "result" : "updated",
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 4,
  "_primary_term" : 1
}
```

查询该文档之后：

![image-20211211192502822](http://badwomen.asia/image-20211211192502822.png)



> 第二种方式

前面是一样的插入一条数据：

```json
POST /products/_doc
{
  "title":"Iphone 4",
  "created_at": "2014-09-01",
  "price": "499",
  "description": "这是一部过时了的Iphone 4手机"
}



{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "bbw6qX0BViY44oHwSzKn",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 3,
  "_primary_term" : 1
}

```

```json
# 第二种方式更新 在原来的文档上进行更新
POST /products/_doc/brw9qX0BViY44oHwbDLo/_update 
{
 "doc": {
  "title": "Iphone15",
  "price": "4000"
 }
}
```



![image-20211211192837439](http://badwomen.asia/image-20211211192837439.png)

数据更新完成，原来的数据依然存在。

#### 批量操作

#### 批量操作

```http
POST /products/_doc/_bulk #批量索引两条文档
 	{"index":{"_id":"1"}}
  		{"title":"iphone14","price":8999.99,"created_at":"2021-09-15","description":"iPhone 13屏幕采用6.8英寸OLED屏幕"}
	{"index":{"_id":"2"}}
  		{"title":"iphone15","price":8999.99,"created_at":"2021-09-15","description":"iPhone 15屏幕采用10.8英寸OLED屏幕"}
```

![image-20211211193029750](http://badwomen.asia/image-20211211193029750.png)

```http
POST /products/_doc/_bulk #更新文档同时删除文档
	{"update":{"_id":"1"}}
		{"doc":{"title":"iphone17"}}
	{"delete":{"_id":2}}
	{"index":{}}
		{"title":"iphone19","price":8999.99,"created_at":"2021-09-15","description":"iPhone 19屏幕采用61.8英寸OLED屏幕"}
```

> 说明:批量时不会因为一个失败而全部失败,而是继续执行后续操作,在返回时按照执行的状态返回!

