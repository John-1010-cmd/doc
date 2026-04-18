---
title: Elasticsearch-入门篇
date: 2023-06-07
updated : 2023-06-08
categories: 
- Elasticsearch
tags: 
  - Elasticsearch
  - 实战
description: Elasticsearch 搜索引擎入门与实战，涵盖倒排索引、分词、聚合查询及集群部署。
series: 搜索引擎技术
series_order: 1
---

## 入门

### HTTP

#### 创建索引

```
PUT http://localhost:9200/shopping
```

![image-20230607221035559](image-20230607221035559.png)

重复请求，会返回错误信息。

![image-20230607221151107](image-20230607221151107.png)

#### 查询索引

##### 查看所有索引

```
GET http://localhost:9200/_cat/indices?v
```

|      表头      |                             含义                             |
| :------------: | :----------------------------------------------------------: |
|     health     | 当前服务器健康状态： green(集群完整) yellow(单点正常、集群不完整) red(单点不正常) |
|     status     |                      索引打开、关闭状态                      |
|     index      |                            索引名                            |
|      uuid      |                         索引统一编号                         |
|      pri       |                          主分片数量                          |
|      rep       |                           副本数量                           |
|   docs.count   |                         可用文档数量                         |
|  docs.deleted  |                   文档删除状态（逻辑删除）                   |
|   store.size   |                 主分片和副分片整体占空间大小                 |
| pri.store.size |                       主分片占空间大小                       |

##### 查看单个索引

```
GET http://localhost:9200/shopping
```

![image-20230607220955147](image-20230607220955147.png)

#### 删除索引

```
DELETE http://localhost:9200/shopping
```

#### 创建文档

```
POST http://localhost:9200/shopping/_doc
```

```json
{
    "title":"小米手机",
    "category":"小米",
    "image":"http://xiaomi.com/xm.jpg",
    "price":3999.00
}
```

![image-20230607220536885](image-20230607220536885.png)

#### 查询

##### 主键查询

```
GET http://localhost:9200/shopping/_doc/1
```

![image-20230607221613482](image-20230607221613482.png)

若内容不存在

![image-20230607222014641](image-20230607222014641.png)

##### 全查询

```
GET http://localhost:9200/shopping/_search
```

![image-20230607222424222](image-20230607222424222.png)

#### 全量修改

和新增文档一样，输入相同的 URL 地址请求，如果请求体变化，会将原有的数据内容覆盖

![image-20230607222517456](image-20230607222517456.png)

#### 局部修改

修改数据时，也可以只修改某一给条数据的局部信息

```
POST http://localhost:9200/shopping/_update/1
```

#### 删除

删除一个文档不会立即从磁盘上移除，它只是被标记成已删除（逻辑删除）。

```
DELETE http://localhost:9200/shopping/_doc/1
```

![image-20230607222711064](image-20230607222711064.png)

#### 条件查询

```
GET http://localhost:9200/shopping/_search?q=category:小米
```

上述URL带参数形式查询，这很容易让不善者心怀恶意，或者参数值出现中文会出现乱码情况。为了避免这些情况，我们可用使用带JSON请求体请求进行查询。

```
GET http://localhost:9200/shopping/_search
```

```json
{
    "query":{
        "match":{
            "category":"小米"
        }
    }
}
```

![image-20230607223532015](image-20230607223532015.png)

#### 分页查询

```
GET http://localhost:9200/shopping/_search
```

```json
{
    "query":{
        "match_all":{}
    }，
    "from":0,
    "to":2
}
```

#### 查询排序

```
GET http://localhost:9200/shopping/_search
```

```json
{
    "query":{
        "match_all":{}
    },
    "sort":{
        "price":{
            "order":"desc"
        }
    }
}
```

#### 多条件查询

```
GET http://localhost:9200/shopping/_search
```

```json
{
	"query":{
        "bool":{
            "must":[{
                "match":{
                    "category":"小米"
                }
            },{
                "match":{
                	"price":3999.00
            	}
            }]
        }
    }
}
```

- must相当于数据库的&&
- should相当于数据库的||

#### 范围查询

```
GET http://localhost:9200/shopping/_search
```

```json
{
	"query":{
		"bool":{
			"should":[{
				"match":{
					"category":"小米"
				}
			},{
				"match":{
					"category":"华为"
				}
			}],
            "filter":{
            	"range":{
                	"price":{
                    	"gt":2000
                	}
	            }
    	    }
		}
	}
}
```

#### 全文检索

```
GET http://localhost:9200/shopping/_search
```

```json
{
    "query":{
        "match":{
            "category":"小华"
        }
    }
}
```

返回结果带回品牌有"小米"和"华为"的

#### 完全匹配

```
GET http://localhost:9200/shopping/_search
```

```json
{
	"query":{
		"match_phrase":{
			"category" : "为"
		}
	}
}
```

#### 高亮查询

```
GET http://localhost:9200/shopping/_search
```

```json
{
	"query":{
		"match_phrase":{
			"category" : "米"
		}
	},
    "highlight":{
        "fields":{
            "category":{}//<----高亮这字段
        }
    }
}
```

![image-20230607225608306](image-20230607225608306.png)

#### 聚合查询

```
GET http://localhost:9200/shopping/_search
```

```json
{
	"aggs":{//聚合操作
		"price_group":{//名称，随意起名
			"terms":{//分组
				"field":"price"//分组字段
			}
		}
	}
}
```

返回结果会附带原始数据的。若不想要不附带原始数据的结果，json需改为

```json
{
	"aggs":{//聚合操作
		"price_group":{//名称，随意起名
			"terms":{//分组
				"field":"price"//分组字段
			}
		}
	},
    "size":0
}
```

**求平均值**

```json
{
	"aggs":{
		"price_avg":{//名称，随意起名
			"avg":{//求平均
				"field":"price"
			}
		}
	},
    "size":0
}
```

#### 映射关系

##### 创建索引

```
PUT http://localhost:9200/user
```

##### 创建映射

```
PUT http://localhost:9200/user/_mapping
```

```json
{
    "properties": {
        "name":{
        	"type": "text",
        	"index": true
        },
        "sex":{
        	"type": "keyword",
        	"index": true
        },
        "tel":{
        	"type": "keyword",
        	"index": false
        }
    }
}
```

##### 查询映射

```
GET http://localhost:9200/user/_mapping
```

##### 增加数据

```
PUT http://localhost:9200/user/_create/1001
```

```json
{
    "name":"小米",
    "sex":"男",
    "tel":"1111"
}
```

##### 查找

```
GET http://localhost:9200/user/_search
```

```json
{
    "query":{
        "match":{
            "name":"小"
        }
    }
}
```

### JavaAPI

#### 创建索引

```java
// 创建客户端对象RestHighLevelClient 
// 创建索引请求对象
 CreateIndexRequest request = new CreateIndexRequest("user2");
// 发送请求 获取响应
```

#### 查询索引

```java
// 查询索引，请求对象
GetIndexRequest request = new GetIndexRequest("user2");
// 发送请求 获取响应
GetIndexResponse response = client.indices().get(request, RequestOptions.DEFAULT);
```

#### 删除索引

```java
// 删除索引 - 请求对象
DeleteIndexRequest request = new DeleteIndexRequest("user2");
// 发送请求，获取响应
AcknowledgedResponse response = client.indices().delete(request,RequestOptions.DEFAULT);
```

#### 新增文档

```java
RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost", 9200, "http")));
// 新增文档 - 请求对象
IndexRequest request = new IndexRequest();
request.index("user").id("1001");
User user = new User("zhangsan",25,"男");
String productJson = new ObjectMapper().writeValueAsString(user);
request.source(productJson,XContentType.JSON);
IndexResponse response = client.index(request, RequestOptions.DEFAULT);
client.close();
```

#### 修改文档

```java
UpdateRequest request = new UpdateRequest();
request.index("user").id("1001");
request.doc(XContentType.JSON, "sex", "女");
UpdateResponse response = client.update(request, RequestOptions.DEFAULT);
```

#### 查询文档

```java
GetRequest request = new GetRequest().index("user").id("1001");
GetResponse response = client.get(request, RequestOptions.DEFAULT);
```

#### 删除文档

```java
DeleteRequest request = new DeleteRequest().index("user").id("1001");
DeleteResponse response = client.delete(request, RequestOptions.DEFAULT);
```

#### 批量新增文档

```java
BulkRequest request = new BulkRequest();
request.add(new IndexRequest().index("user").id("1001").source(XContentType.JSON, "name", "zhangsan"));
request.add(new IndexRequest().index("user").id("1002").source(XContentType.JSON, "name", "lisi"));
request.add(new IndexRequest().index("user").id("1003").source(XContentType.JSON, "name", "wangwu"));
BulkResponse responses = client.bulk(request, RequestOptions.DEFAULT);
```

#### 批量删除文档

```java
BulkRequest request = new BulkRequest();
request.add(new DeleteRequest().index("user").id("1001"));
request.add(new DeleteRequest().index("user").id("1002"));
request.add(new DeleteRequest().index("user").id("1003"));
BulkResponse responses = client.bulk(request, RequestOptions.DEFAULT);
```

#### 高级查询 - 全量查询

```java
SearchRequest request = new SearchRequest();
request.indices("user");
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
sourceBuilder.query(QueryBuilders.matchAllQuery());
request.source(sourceBuilder);
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
```

#### 高级查询 - 条件查询

```java
SearchRequest request = new SearchRequest();
request.indices("user");
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
sourceBuilder.query(QueryBuilders.termQuery("age", "30"));
request.source(sourceBuilder);
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
```

#### 高级查询 - 分页查询

```java
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
sourceBuilder.query(QueryBuilders.matchAllQuery());
sourceBuilder.from(0);
sourceBuilder.size(2);
request.source(sourceBuilder);
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
```

#### 高级查询 - 查询排序

```java
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
sourceBuilder.query(QueryBuilders.matchAllQuery());
sourceBuilder.sort("age", SortOrder.ASC);
request.source(sourceBuilder);
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
```

#### 高级查询 - 组合查询

```java
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
// 必须包含
boolQueryBuilder.must(QueryBuilders.matchQuery("age", "30"));
// 一定不含
boolQueryBuilder.mustNot(QueryBuilders.matchQuery("name", "zhangsan"));
// 可能包含
boolQueryBuilder.should(QueryBuilders.matchQuery("sex", "男"));
sourceBuilder.query(boolQueryBuilder);
request.source(sourceBuilder);
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
```

#### 高级查询 - 范围查询

```java
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
RangeQueryBuilder rangeQuery = QueryBuilders.rangeQuery("age");
rangeQuery.lte("40");
sourceBuilder.query(rangeQuery);
request.source(sourceBuilder);
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
```

#### 高级查询 - 模糊查询

```java
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
sourceBuilder.query(QueryBuilders.fuzzyQuery("name","wangwu").fuzziness(Fuzziness.ONE));
request.source(sourceBuilder);
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
```

#### 高级查询 - 高亮查询

```java
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
TermsQueryBuilder termsQueryBuilder = QueryBuilders.termsQuery("name","zhangsan");
sourceBuilder.query(termsQueryBuilder);
HighlightBuilder highlightBuilder = new HighlightBuilder();
highlightBuilder.preTags("<font color='red'>");//设置标签前缀
highlightBuilder.postTags("</font>");//设置标签后缀
highlightBuilder.field("name");//设置高亮字段
sourceBuilder.highlighter(highlightBuilder);
request.source(sourceBuilder);
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
```

#### 高级查询 - 最大值查询

```java
SearchRequest request = new SearchRequest().indices("user");
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
sourceBuilder.aggregation(AggregationBuilders.max("maxAge").field("age"));
request.source(sourceBuilder);
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
```

#### 高级查询 - 分组查询

```java
SearchRequest request = new SearchRequest().indices("user");
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
sourceBuilder.aggregation(AggregationBuilders.terms("age_groupby").field("age"));
request.source(sourceBuilder);
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
```

---

## Elasticsearch 核心原理

### 倒排索引

Elasticsearch 使用倒排索引（Inverted Index）实现快速全文检索。与传统数据库按行存储不同，倒排索引将文档中的每个词映射到包含该词的文档列表。

**正排索引 vs 倒排索引**

| 类型 | 结构 | 适用场景 |
|------|------|----------|
| 正排索引 | 文档 ID → 文档内容 | 按 ID 查询 |
| 倒排索引 | 词项 → 文档 ID 列表 | 全文检索 |

**倒排索引核心组成**：
1. **Term Dictionary（词项字典）**：所有分词后的词项有序列表
2. **Posting List（倒排列表）**：记录每个词项出现的文档 ID 列表
3. **Term Index（词项索引）**：对 Term Dictionary 建立的索引，加速词项查找

### 分词原理

分词（Analysis）是将文本切分为可搜索词项的过程，由以下组件完成：

- **Character Filter**：预处理原始文本（如去除 HTML 标签、替换特殊字符）
- **Tokenizer**：将文本切分为词项（如标准分词器按空格和标点切分）
- **Token Filter**：对词项进行后处理（如小写转换、停用词过滤、同义词扩展）

**常用中文分词器**：
- **IK 分词器**：支持细粒度和智能分词两种模式，适合中文搜索
- **jieba 分词**：基于概率模型的中文分词，支持自定义词典

### 为什么 Elasticsearch 快

1. **倒排索引**：O(1) 词项定位，避免全表扫描
2. **压缩算法**：Posting List 使用 FOR + RBM 压缩，减少存储和 I/O
3. **内存缓存**：频繁访问的索引段缓存在内存中（File System Cache）
4. **并行查询**：单个查询可在多个分片上并行执行
5. **近实时**：写入数据先存于内存缓冲区（refresh_interval 默认 1s 刷盘）

---

## 集群架构

### 节点类型

| 节点类型 | 职责 | 生产建议 |
|----------|------|----------|
| Master | 管理集群元数据、索引创建、分片分配 | 至少 3 个候选节点，专用 master 不存数据 |
| Data | 存储分片数据，执行 CRUD 和聚合操作 | 根据数据量横向扩展 |
| Coordinating | 接收客户端请求，分发到数据节点，聚合结果 | 高并发场景可独立部署 |
| Ingest | 预处理文档（如分词、字段转换） | 大量写入前置处理时使用 |

### 分片与副本

- **主分片（Primary Shard）**：数据写入时分配到主分片，数量在索引创建时固定，不可修改
- **副本分片（Replica Shard）**：主分片的完整拷贝，提供读请求负载均衡和故障恢复

**分片规划原则**：
- 单个分片大小控制在 20-50 GB
- 避免过多小分片（每个分片有额外内存开销）
- 副本数 ≥ 1 保证高可用，写入压力大时可临时设为 0

### 写入流程

1. 客户端向协调节点发送写入请求
2. 协调节点根据 `_routing` 计算目标主分片
3. 主分片执行写入，写入 Translog（保证持久性）
4. 主分片同步写入到所有副本分片
5. 所有副本确认后，返回成功响应给客户端
6. 内存缓冲区定期 refresh 到文件系统缓存（默认 1s），生成新段（segment）
7. 段文件定期合并（merge），并 flush 到磁盘

---

## 数据类型

| 类型分类 | 具体类型 | 说明 |
|----------|----------|------|
| 字符串 | `text`、`keyword` | text 会被分词，keyword 不分词用于精确匹配和聚合 |
| 数值 | `long`、`integer`、`short`、`byte`、`double`、`float` | 根据数值范围选择 |
| 日期 | `date` | 支持多种日期格式，内部存储为毫秒时间戳 |
| 布尔 | `boolean` | true / false |
| 二进制 | `binary` | Base64 编码的字符串 |
| 范围 | `integer_range`、`float_range`、`date_range` | 存储范围值 |
| 对象 | `object`、`nested` | object 内部字段扁平化可能导致数组交叉匹配，nested 保持对象独立性 |
| 地理 | `geo_point`、`geo_shape` | 地理位置坐标和几何图形 |

---

## 优化建议

### 索引设计
- 合理设置分片数和副本数，避免频繁 rebalance
- 对不需要全文检索的字段使用 `keyword` 类型
- 对不需要聚合/排序的字段设置 `"index": false`
- 禁用 `_all` 字段（ES 7+ 已默认禁用）

### 写入优化
- 批量写入（Bulk API）减少网络往返
- 调整 `refresh_interval`（日志场景可设为 30s）
- 写入高峰期可临时减少副本数

### 查询优化
- 避免深度分页，使用 `search_after` 或滚动查询
- 过滤条件优先使用 `filter` 上下文（不计算相关度，可缓存）
- 对聚合查询启用 `eager_global_ordinals` 预加载

### 运维建议
- 监控集群健康状态（green / yellow / red）
- 定期清理过期索引（ILM 索引生命周期管理）
- JVM 堆内存不超过 32 GB（避免指针压缩失效）

