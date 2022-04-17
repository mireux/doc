# SpringBoot整合ElasticSearch

## 引入依赖

pom.xml

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
```

## 配置客户端

```java
package com.example.springbootelastcsearch.config;

import org.elasticsearch.client.RestHighLevelClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.elasticsearch.client.ClientConfiguration;
import org.springframework.data.elasticsearch.client.RestClients;
import org.springframework.data.elasticsearch.config.AbstractElasticsearchConfiguration;

@Configuration
public class RestClientConfig extends AbstractElasticsearchConfiguration {
	
    @Value("${ESHost}")
    private String host;	// 为了后期好更改es配置 host为127.0.0.1:9200

    @Override
    public RestHighLevelClient elasticsearchClient() {
        final ClientConfiguration clientConfiguration = ClientConfiguration.builder()
                .connectedTo(host)
                .build();
        return RestClients.create(clientConfiguration).rest();
    }
}

```

## 客户端对象

当配置完客户端连接设置之后，SpringBoot会自动注入两个对象，分别是

- ElasticsearchOperations 
  - 是一种面向对象方式操作ES
- RestHighLevelClient **推荐**
  - 更接近之前的kibana操作ES的方式

### ElasticsearchOperations

- 特点: 始终使用面向对象方式操作 ES
  - 索引: 用来存放相似文档集合
  - 映射: 用来决定放入文档的每个字段以什么样方式录入到 ES 中 字段类型 分词器..
  - 文档: 可以被索引最小单元  json 数据格式

#### 相关注解

```java
@Document(indexName = "testproducts", createIndex = true)
@Data
public class Product {
		@Id
    private Integer id;
    @Field(type = FieldType.Keyword)
    private String title;
    @Field(type = FieldType.Float)
    private Double price;
    @Field(type = FieldType.Text)
    private String description;
}
//1. @Document(indexName = "products", createIndex = true) 用在类上 作用:代表一个对象为一个文档
		-- indexName属性: 创建索引的名称
    -- createIndex属性: 是否创建索引 默认为true
//2. @Id 用在属性上  作用:将对象id字段与ES中文档的_id对应
//3. @Field(type = FieldType.Keyword) 用在属性上 作用:用来描述属性在ES中存储类型以及分词情况
    -- type: 用来指定字段类型
```

#### 创建文档

一下操作都是在Test中完成

> 第一个展示全部程序 后面只展示接口程序

```java
package com.example.springbootelastcsearch;


import com.example.springbootelastcsearch.entity.Product;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;

public class TestElasticSearchOperations extends SpringbootElastcsearchApplicationTests {

    private final ElasticsearchOperations elasticsearchOperations;

    @Autowired
    public TestElasticSearchOperations(ElasticsearchOperations elasticsearchOperations) {
        this.elasticsearchOperations = elasticsearchOperations;
    }

    @Test
    public void TestCreate() {
        Product product = new Product();
        product.setId(1);
        product.setTitle("矿泉水");
        product.setPrice(2.0);
        product.setDescription("矿泉水真的好喝呢");
        Product save = elasticsearchOperations.save(product);
        System.out.println(save);
    }
}

```

![image-20211212153733961](http://badwomen.asia/image-20211212153733961.png)

![image-20211212153755397](http://badwomen.asia/image-20211212153755397.png)

添加成功！

#### 删除文档

```java
    // 删除索引
    @Test
    public void TestDelete() {
        Product product = new Product();
        product.setId(1);
        String delete = elasticsearchOperations.delete(product);
        System.out.println(delete);
    }
```



![image-20211212154104124](http://badwomen.asia/image-20211212154104124.png)

![image-20211212154114330](http://badwomen.asia/image-20211212154114330.png)

#### 查询文档

>  先重新加入一条数据

```java
    //查询文档
    @Test
    public void TestQuery() {
        Product product = elasticsearchOperations.get("1", Product.class);
        System.out.println(product);
    }

```

![image-20211212154422995](http://badwomen.asia/image-20211212154422995.png)



#### 更新文档

```java
   // 更新文档
    @Test
    public void TestUpdate() {
        Product product = new Product();
        product.setId(1);
        product.setTitle("怡宝矿泉水");
        product.setPrice(129.11);
        product.setDescription("我们喜欢喝矿泉水,你们喜欢吗....");
        Product save = elasticsearchOperations.save(product);//不存在添加,存在更新
        System.out.println(save);
    }

```

![image-20211212154631373](http://badwomen.asia/image-20211212154631373.png)



#### 查询所有

```java
    // 查询所有
    @Test
    public void TestQueryAll() {
        SearchHits<Product> productSearchHits = elasticsearchOperations.search(Query.findAll(), Product.class);
        productSearchHits.forEach(productSearchHit -> {
            Product content = productSearchHit.getContent();
            System.out.println(content);
        });
    }
}
```

![image-20211212154948470](http://badwomen.asia/image-20211212154948470.png)

#### 删除所有

```java
@Test
public void testDeleteAll() {
  elasticsearchOperations.delete(Query.findAll(), Product.class);
}
```





### RestHighLevelClient

实际上好像是更常用的是RestHighLevelClient，因为它的操作更接近kibana，能够进行更加复杂的操作。

#### 创建索引映射

```java
    @Test
	public void testCreateIndex() throws IOException {
        CreateIndexRequest createIndexRequest = new CreateIndexRequest("testhighlevel");
        createIndexRequest.mapping("{\n" +
                "    \"properties\": {\n" +
                "      \"title\":{\n" +
                "        \"type\": \"keyword\"\n" +
                "      },\n" +
                "      \"description\":{\n" +
                "        \"type\": \"text\"\n" +
                "      },\n" +
                "      \"price\": {\n" +
                "        \"type\": \"float\"\n" +
                "      }\n" +
                "    }\n" +
                "  }", XContentType.JSON);
        CreateIndexResponse createIndexResponse = restHighLevelClient.indices().create(createIndexRequest, RequestOptions.DEFAULT);
        System.out.println(createIndexResponse.isAcknowledged());
        restHighLevelClient.close();
    }
```

![image-20211212160507690](http://badwomen.asia/image-20211212160507690.png)

![image-20211212160615981](http://badwomen.asia/image-20211212160615981.png)



#### 索引文档

```java
 /**
     * 创建文档
     * @throws IOException
     */
    @Test
public void testIndex() throws IOException {
        IndexRequest indexRequest = new IndexRequest("testhighlevel");
        indexRequest.id("1").source("{\n" +
                "  \"title\": \"黄焖鸡米饭\",\n" +
                "  \"price\": 20.0,\n" +
                "  \"description\":\"黄焖鸡米饭真的好吃\"\n" +
                "}", XContentType.JSON);
        IndexResponse indexResponsee = restHighLevelClient.index(indexRequest, RequestOptions.DEFAULT);
        System.out.println(indexResponsee.status());
    }
```

![image-20211212162635310](http://badwomen.asia/image-20211212162635310.png)

![image-20211212162648976](http://badwomen.asia/image-20211212162648976.png)

#### 更新文档

```java
    /**
     * 更新文档
     * @throws IOException
     */
    @Test
	public void TestUpdate() throws IOException {

        UpdateRequest updateRequest = new UpdateRequest("testhighlevel","1");
        updateRequest.doc(" {\n" +
                "    \"price\": 20.0\n" +
                "  }",XContentType.JSON);
        restHighLevelClient.update(updateRequest,RequestOptions.DEFAULT);
    }
```

![image-20211212163211055](http://badwomen.asia/image-20211212163211055.png)





#### 删除文档

```java
@Test
public void testDelete() throws IOException {
  DeleteRequest deleteRequest = new DeleteRequest("testhighlevel","1");
  DeleteResponse delete = restHighLevelClient.delete(deleteRequest, RequestOptions.DEFAULT);
  System.out.println(delete.status());
}
```

#### 基于id查询文档

```java
    /**
     * 基于id查询
     * @throws IOException
     */
    @Test
    public void TestQueryById() throws IOException {
        GetRequest getRequest = new GetRequest("testhighlevel","1");
        GetResponse documentFields = restHighLevelClient.get(getRequest, RequestOptions.DEFAULT);
        System.out.println(documentFields.getSourceAsString());
    }

```

![image-20211212163452032](http://badwomen.asia/image-20211212163452032.png)

#### 查询所有

```java
    /**
     * 查询所有
     * @throws IOException
     */
    @Test
    public void TestQueryAll() throws IOException {
        SearchRequest searchRequest = new SearchRequest("testhighlevel");
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.query(QueryBuilders.matchAllQuery());
        searchRequest.source(sourceBuilder);
        SearchResponse search = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        SearchHit[] hits = search.getHits().getHits();
        for (SearchHit hit : hits) {
            System.out.println(hit.getSourceAsString());
        }

    }
```

![image-20211212163851341](http://badwomen.asia/image-20211212163851341.png)

#### 综合查询

```java
  // 抽象出查询方法
        public void query(QueryBuilder queryBuilder) throws IOException {
            SearchRequest searchRequest = new SearchRequest("testhighlevel");
            SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
            sourceBuilder.query(queryBuilder);// 指定查询条件 
            searchRequest.source(sourceBuilder);
            SearchResponse search = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
            SearchHit[] hits = search.getHits().getHits();
            for (SearchHit hit : hits) {
                System.out.println(hit.getSourceAsString());
            }
        }

```



```java
        @Test
        public void TestQueryTerm() throws IOException {
//             关键字查询
//            query(QueryBuilders.termQuery("description","米饭"));
//            范围查询
//            query(QueryBuilders.rangeQuery("price").gte(15).lte(20));
//            多字段查询
//            query(QueryBuilders.multiMatchQuery("米饭","title","description"));
        }
```

剩下的就不都展示了，其实跟kibana中都差不多。





#### 高亮查询

```java
 /**
     * 高亮查询
     * @throws IOException
     */
    @Test
    public void TestHighLight() throws IOException {
        SearchRequest searchRequest = new SearchRequest("products");
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        highlightBuilder.requireFieldMatch(false).field("description").field("title").preTags("<span style='color:red'>").postTags("</span>");
        SearchSourceBuilder highlighter = searchSourceBuilder.query(QueryBuilders.multiMatchQuery("iphone15", "description", "title")).highlighter(highlightBuilder);
        searchRequest.source(highlighter);
        SearchResponse search = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        System.out.println(search.getHits().getTotalHits());
        SearchHit[] hits = search.getHits().getHits();
        for (SearchHit hit : hits) {
            Map<String, HighlightField> highlightFields = hit.getHighlightFields();
            if(highlightFields.containsKey("description")) {
                System.out.println("description：" + highlightFields.get("description").getFragments()[0]);
            }
            if(highlightFields.containsKey("title")) {
                System.out.println("title：" + highlightFields.get("title").getFragments()[0]);
            }
            System.out.println(hit.getSourceAsString());
        }
    }
```



![image-20211212231820934](http://badwomen.asia/image-20211212231820934.png)





#### 按照字段排序

```java
    /**
     * 按照字段排序
     * @throws IOException
     */
    @Test
	public void TestSort() throws IOException {
        SearchRequest searchRequest = new SearchRequest("products");
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        searchSourceBuilder.query(QueryBuilders.matchAllQuery()).sort("price",SortOrder.DESC);
        searchRequest.source(searchSourceBuilder);
        SearchResponse search = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        SearchHit[] hits = search.getHits().getHits();
        for (SearchHit hit : hits) {
            System.out.println("hit = " + hit.getSourceAsString());
        }
    }
```

![image-20211212232529984](http://badwomen.asia/image-20211212232529984.png)



#### 返回指定字段

```java
    /**
     * 分页查询
     * @throws IOException
     */
    @Test
    public void TestReturnField() throws IOException {
        SearchRequest searchRequest = new SearchRequest("products");
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        searchSourceBuilder.query(QueryBuilders.matchAllQuery())
                .from(0)
                .size(2);
        searchRequest.source(searchSourceBuilder);
        SearchResponse search = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        SearchHit[] hits = search.getHits().getHits();
        for (SearchHit hit : hits) {
            System.out.println("hit = " + hit.getSourceAsString());
        }
    }

```

![image-20211212232933857](http://badwomen.asia/image-20211212232933857.png)





#### 过滤查询

```java
    /**
     * 过滤查询
     */
    @Test
    public void TestFilterQuery() throws IOException {
        SearchRequest searchRequest = new SearchRequest("products");
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        searchSourceBuilder
                .query(QueryBuilders.matchAllQuery())
                .postFilter(QueryBuilders.termQuery("description","这"));
        searchRequest.source(searchSourceBuilder);
        SearchResponse search = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        SearchHit[] hits = search.getHits().getHits();
        for (SearchHit hit : hits) {
            System.out.println(hit.getSourceAsString());
        }
    }
```

![image-20211212233437530](http://badwomen.asia/image-20211212233437530.png)





### **聚合查询**

聚合：**英文为Aggregation，是es除搜索功能外提供的针对es数据做统计分析的功能**。聚合有助于根据搜索查询提供聚合数据。聚合查询是数据库中重要的功能特性，ES作为搜索引擎兼数据库，同样提供了强大的聚合分析能力。它基于查询条件来对数据进行分桶、计算的方法。有点类似于 SQL 中的 group by 再加一些函数方法的操作。

> `注意事项：text类型是不支持聚合的。`

**测试数据**

```json
# 创建索引 index 和映射 mapping
PUT /fruit
{
  "mappings": {
    "properties": {
      "title":{
        "type": "keyword"
      },
      "price":{
        "type":"double"
      },
      "description":{
        "type": "text",
        "analyzer": "ik_max_word"
      }
    }
  }
}
# 放入测试数据
PUT /fruit/_bulk
{"index":{}}
  {"title" : "面包","price" : 19.9,"description" : "小面包非常好吃"}
{"index":{}}
  {"title" : "旺仔牛奶","price" : 29.9,"description" : "非常好喝"}
{"index":{}}
  {"title" : "日本豆","price" : 19.9,"description" : "日本豆非常好吃"}
{"index":{}}
  {"title" : "小馒头","price" : 19.9,"description" : "小馒头非常好吃"}
{"index":{}}
  {"title" : "大辣片","price" : 39.9,"description" : "大辣片非常好吃"}
{"index":{}}
  {"title" : "透心凉","price" : 9.9,"description" : "透心凉非常好喝"}
{"index":{}}
  {"title" : "小浣熊","price" : 19.9,"description" : "童年的味道"}
{"index":{}}
  {"title" : "海苔","price" : 19.9,"description" : "海的味道"}
```

#### 使用

其实跟mysql中的group by之后使用函数差不多

#### 根据某个字段分组

```http
# 根据某个字段进行分组 统计数量
GET /fruit/_search
{
  "query": {
    "term": {
      "description": {
        "value": "好吃"
      }
    }
  }, 
  "aggs": {
    "price_group": {
      "terms": {
        "field": "price"
      }
    }
  }
}
```

#### 求最大值

```http
# 求最大值 
GET /fruit/_search
{
  "aggs": {
    "price_max": {
      "max": {
        "field": "price"
      }
    }
  }
  }
```

#### 求最小值

```http
# 求最小值
GET /fruit/_search
{
  "aggs": {
    "price_min": {
      "min": {
        "field": "price"
      }
    }
  }
  }
```

#### 求平均值

```http
# 求平均值
GET /fruit/_search
{
  "aggs": {
    "price_agv": {
      "avg": {
        "field": "price"
      }
    }
  }
  }
```

#### 求和

```http
# 求和
GET /fruit/_search
{
  "aggs": {
    "price_sum": {
      "sum": {
        "field": "price"
      }
    }
  }
  }
```

#### 整合应用

#### 

```java
// 求不同价格的数量
@Test
public void testAggsPrice() throws IOException {
  SearchRequest searchRequest = new SearchRequest("fruit");
  SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
  sourceBuilder.aggregation(AggregationBuilders.terms("group_price").field("price"));
  searchRequest.source(sourceBuilder);
  SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
  Aggregations aggregations = searchResponse.getAggregations();
  ParsedDoubleTerms terms = aggregations.get("group_price");
  List<? extends Terms.Bucket> buckets = terms.getBuckets();
  for (Terms.Bucket bucket : buckets) {
    System.out.println(bucket.getKey() + ", "+ bucket.getDocCount());
  }
}
```

![image-20211213152412366](http://badwomen.asia/image-20211213152412366.png)

```java
// 求不同名称的数量
@Test
public void testAggsTitle() throws IOException {
  SearchRequest searchRequest = new SearchRequest("fruit");
  SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
  sourceBuilder.aggregation(AggregationBuilders.terms("group_title").field("title"));
  searchRequest.source(sourceBuilder);
  SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
  Aggregations aggregations = searchResponse.getAggregations();
  ParsedStringTerms terms = aggregations.get("group_title");
  List<? extends Terms.Bucket> buckets = terms.getBuckets();
  for (Terms.Bucket bucket : buckets) {
  	System.out.println(bucket.getKey() + ", "+ bucket.getDocCount());
  }
}
```

![image-20211213152745943](http://badwomen.asia/image-20211213152745943.png)

```java
// 求和
@Test
public void testAggsSum() throws IOException {
  SearchRequest searchRequest = new SearchRequest("fruit");
  SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
  sourceBuilder.aggregation(AggregationBuilders.sum("sum_price").field("price"));
  searchRequest.source(sourceBuilder);
  SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
  ParsedSum parsedSum = searchResponse.getAggregations().get("sum_price");
  System.out.println(parsedSum.getValue());
}
```

![image-20211213153626368](http://badwomen.asia/image-20211213153626368.png)



## 后记

我的ES学习日记到这就差不多了 在底层的东西说实话真来不太及学习了，对于我而言ES的要求就是掌握即可。

