---
date: 2026-07-17
tags: [黑马商城微服务, Elasticsearch, DSL, RestClient]
---

# ES搜索：DSL查询语法与RestClient实现

## 一、写在前面

上一篇我们学了 ES 的索引库和文档 CRUD，数据已经导入了。但搜索还是根据 id 查，不是真正的"搜索"。

今天学 ES 的核心能力：**基于 DSL 的全文搜索**。学完你能掌握：
1. 用 DSL 查询语法写出各种搜索条件
2. 用 Java RestClient 对应写出同样的查询
3. 高亮、聚合等进阶功能

## 二、前置知识

- [[ES入门与IK分词器]]：索引库、分词、倒排索引
- [[RestClient操作ES：索引库与文档CRUD]]：RestHighLevelClient 基本用法
- 已有 `items` 索引库，字段：name(text)、brand(keyword)、category(keyword)、price(integer)、sold(integer)、isAD(boolean)、updateTime(date)

---

## 三、DSL 查询结构总览

ES 查询请求体就四块：

```json
{
  "query": { ... },       // 查询条件
  "sort": [ ... ],        // 排序
  "from": 0, "size": 10,  // 分页
  "highlight": { ... },   // 高亮
  "aggs": { ... }         // 聚合
}
```

对应 Java API：

```java
request.source()
    .query(...)         // query
    .sort(...)          // sort
    .from(0).size(10)   // from/size
    .highlighter(...)   // highlight
    .aggregation(...)   // aggs
```

---

## 四、查询分类

ES 查询分两大类：

| 类型 | 说明 | 例子 |
|---|---|---|
| **叶子查询** | 在特定字段查特定值 | match、term、range |
| **复合查询** | 组合多个叶子查询 | bool、function_score |

叶子查询又细分：
- **全文检索**：会对搜索词分词（match、multi_match）→ 适合 text 字段
- **精确查询**：不分词，原样匹配（term、range）→ 适合 keyword/数值/日期/布尔
- **地理坐标查询**：geo_bounding_box、geo_distance

> **核心原则**：字段存的时候分了词（text），查的时候用 match；字段没分词（keyword），查的时候用 term。搞反了搜不到。

---

## 五、叶子查询

### 5.1 match — 模糊搜索

对搜索词分词后匹配，适合 text 字段。

```json
// DSL
{ "match": { "name": "华为手机" } }
```
```java
// Java API
QueryBuilders.matchQuery("name", "华为手机")
```

"华为手机"会被 IK 分词器分成"华为"+"手机"，两个词条都会去匹配。

### 5.2 multi_match — 多字段模糊搜索

一个关键字同时搜多个字段，任一字段命中就算匹配。

```json
// DSL
{ "multi_match": { "query": "手机", "fields": ["name", "category"] } }
```
```java
// Java API
QueryBuilders.multiMatchQuery("手机", "name", "category")
```

### 5.3 term — 精确匹配

不分词，原样匹配。只能查 keyword、数值、日期、布尔类型字段。

```json
// DSL
{ "term": { "brand": { "value": "华为" } } }
```
```java
// Java API
QueryBuilders.termQuery("brand", "华为")
```

> **坑**：用 term 搜 text 字段会搜不到！因为 text 字段存的时候分了词，term 不分词去匹配，对不上。

### 5.4 range — 范围查询

```json
// DSL
{ "range": { "price": { "gte": 1000, "lte": 5000 } } }
```
```java
// Java API
QueryBuilders.rangeQuery("price").gte(1000).lte(5000)
```

四个边界操作符：`gte`(≥)、`gt`(>)、`lte`(≤)、`lt`(<)

### 5.5 match_all — 查询全部

```json
{ "match_all": {} }
```
```java
QueryBuilders.matchAllQuery()
```

### 5.6 叶子查询速查表

| 查询 | DSL | Java API | 场景 |
|---|---|---|---|
| match | `{"match":{"name":"手机"}}` | `matchQuery("name","手机")` | 全文搜索 |
| multi_match | `{"multi_match":{"query":"手机","fields":["name","category"]}}` | `multiMatchQuery("手机","name","category")` | 多字段搜索 |
| term | `{"term":{"brand":{"value":"华为"}}}` | `termQuery("brand","华为")` | 精确匹配 |
| range | `{"range":{"price":{"gte":1000}}}` | `rangeQuery("price").gte(1000)` | 范围查询 |
| match_all | `{"match_all":{}}` | `matchAllQuery()` | 查全部 |

---

## 六、复合查询

### 6.1 bool 查询（最核心）

用逻辑关系组合多个叶子查询：

```json
{
  "bool": {
    "must": [ {"match": {"name": "手机"}} ],       // 必须满足，参与算分
    "filter": [                                      // 必须满足，不参与算分
      {"term": {"brand": "华为"}},
      {"range": {"price": {"gte": 90000, "lte": 159900}}}
    ],
    "should": [ ... ],    // 可选，满足加分
    "must_not": [ ... ]   // 必须不满足
  }
}
```

```java
// Java API
QueryBuilders.boolQuery()
    .must(QueryBuilders.matchQuery("name", "手机"))
    .filter(QueryBuilders.termQuery("brand", "华为"))
    .filter(QueryBuilders.rangeQuery("price").gte(90000).lte(159900))
```

**四种子句速记**：

| 子句 | 逻辑 | 算分 | 用途 |
|---|---|---|---|
| must | AND | ✅ 参与 | 用户搜索关键字 |
| filter | AND | ❌ 不参与 | 品牌/分类/价格过滤 |
| should | OR | ✅ 参与 | 可选条件 |
| must_not | NOT | ❌ 不参与 | 排除条件 |

> **性能原则**：与搜索关键字无关的查询用 filter，不算分，性能更好。

### 6.2 function_score 查询（选讲）

控制相关性算分，实现竞价排名。

```json
{
  "function_score": {
    "query": { "match": { "name": "手机" } },   // 原始查询
    "functions": [{
      "filter": { "term": { "isAD": true } },   // 过滤条件：广告商品
      "weight": 10                                // 权重 ×10
    }],
    "boost_mode": "multiply"                      // 运算模式：相乘
  }
}
```

```java
// Java API
QueryBuilders.functionScoreQuery(
    QueryBuilders.matchQuery("name", "手机"),
    new FunctionScoreQueryBuilder.FilterFunctionBuilder[]{
        new FunctionScoreQueryBuilder.FilterFunctionBuilder(
            QueryBuilders.termQuery("isAD", true),
            ScoreFunctionBuilders.weightFactorFunction(10)
        )
    }
).boostMode(FunctionScoreQueryBoostMode.MULTIPLY)
```

**流程**：原始查询算分 → 过滤广告文档 → 广告文档分数 ×10 → 广告排在前面

四种算分函数：weight(常量)、field_value_factor(字段值)、random_score(随机)、script_score(自定义脚本)

---

## 七、排序

```json
// DSL — 按价格降序
{ "sort": [{ "price": { "order": "desc" } }] }
```
```java
// Java API
.sort("price", SortOrder.DESC)
```

多字段排序：先按价格升序，价格相同再按销量降序

```java
.sort("price", SortOrder.ASC)
.sort("sold", SortOrder.DESC)
```

> **限制**：text 类型字段不能排序，keyword/数值/日期/布尔可以。

---

## 八、分页

### 8.1 基础分页

```json
{ "from": 0, "size": 10 }
```
```java
.from((pageNo - 1) * pageSize)
.size(pageSize)
```

ES 默认 size=10，不写分页只能看到 10 条。

### 8.2 深度分页问题

ES 数据分片存储，查深页码时每个分片都要返回大量数据汇总排序，内存压力大。

**ES 限制**：`from + size > 10000` 会报错。

三种分页方案：

| 方案 | 原理 | 适用场景 |
|---|---|---|
| from + size | 偏移量分页 | 正常分页（前几页） |
| search_after | 从上次排序值继续 | 深度分页、无限滚动 |
| scroll | 快照分页 | 已不推荐 |

> 实际项目限制分页深度就够了（百度最多 77 页，京东最多 100 页）。

---

## 九、高亮

搜索结果中的关键字加 `<em>` 标签，前端加 CSS 样式显示红色。

### 9.1 DSL

```json
{
  "query": { "match": { "name": "手机" } },
  "highlight": {
    "fields": { "name": {} },
    "pre_tags": "<em>",
    "post_tags": "</em>"
  }
}
```

### 9.2 Java API

```java
// 构建高亮条件
request.source().highlighter(
    SearchSourceBuilder.highlight()
        .field("name")
        .preTags("<em>")
        .postTags("</em>")
);
```

### 9.3 解析高亮结果

```java
for (SearchHit hit : response.getHits().getHits()) {
    // 原始数据
    String source = hit.getSourceAsString();
    ItemDoc item = JSONUtil.toBean(source, ItemDoc.class);
    
    // 高亮结果
    Map<String, HighlightField> hfs = hit.getHighlightFields();
    if (CollUtils.isNotEmpty(hfs)) {
        HighlightField hf = hfs.get("name");
        if (hf != null) {
            String hfName = hf.getFragments()[0].string();
            item.setName(hfName);  // 用高亮结果替换原始 name
        }
    }
}
```

### 9.4 高亮三要素

| 要点 | 说明 |
|---|---|
| 必须有全文检索查询 | match 才分词，才知道哪些词要高亮 |
| 高亮字段必须是 text 类型 | keyword 不分词，没法高亮 |
| 高亮字段默认与搜索字段一致 | 加 `required_field_match=false` 可以不一致 |

---

## 十、聚合

ES 版的 `GROUP BY` + 聚合函数，查询速度快，近实时。

### 10.1 三类聚合

| 类型 | 作用 | SQL 类比 | 例子 |
|---|---|---|---|
| 桶（Bucket） | 分组 | `GROUP BY brand` | 按品牌分组 |
| 度量（Metric） | 计算值 | `AVG(price)` | 平均价格 |
| 管道（Pipeline） | 二次计算 | 嵌套聚合 | 每个品牌的平均价格 |

> **限制**：参与聚合的字段必须是 keyword、数值、日期、布尔类型。text 字段不能聚合。

### 10.2 桶聚合（Bucket）

按品牌分组：

```json
{
  "size": 0,
  "aggs": {
    "brand_agg": {
      "terms": { "field": "brand", "size": 20 }
    }
  }
}
```
```java
request.source().aggregation(
    AggregationBuilders.terms("brand_agg").field("brand").size(20)
);
```

### 10.3 带条件聚合

聚合是对查询结果做聚合，不是全量数据：

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "category": "手机" } },
        { "range": { "price": { "gte": 300000 } } }
      ]
    }
  },
  "size": 0,
  "aggs": {
    "brand_agg": {
      "terms": { "field": "brand", "size": 20 }
    }
  }
}
```

**执行流程**：全部文档 → query 筛选 → aggs 对筛选结果分组

### 10.4 度量聚合（Metric）

桶内嵌套度量聚合，每个桶分别计算：

```json
{
  "aggs": {
    "brand_agg": {
      "terms": { "field": "brand", "size": 20 },
      "aggs": {
        "stats_meric": {
          "stats": { "field": "price" }
        }
      }
    }
  }
}
```
```java
request.source().aggregation(
    AggregationBuilders.terms("brand_agg")
        .field("brand")
        .size(20)
        .subAggregation(
            AggregationBuilders.stats("stats_meric").field("price")
        )
);
```

### 10.5 解析聚合结果

```java
// 获取所有聚合
Aggregations aggregations = response.getAggregations();

// 获取品牌聚合
Terms brandTerms = aggregations.get("brand_agg");

// 遍历桶
for (Terms.Bucket bucket : brandTerms.getBuckets()) {
    String brand = bucket.getKeyAsString();  // 桶的 key（品牌名）
    long count = bucket.getDocCount();       // 桶内文档数
    
    // 取子聚合（度量）
    InternalStats stats = bucket.getAggregations().get("stats_meric");
    double avg = stats.getAvg();
    double min = stats.getMin();
    double max = stats.getMax();
}
```

### 10.6 聚合三要素

| 要素 | 说明 | 例子 |
|---|---|---|
| 聚合名称 | 自定义 key，解析时用 | `"brand_agg"` |
| 聚合类型 | terms/stats/avg/max... | `"terms"` |
| 聚合字段 | 对哪个字段聚合 | `"brand"` |

---

## 十一、全文总结

1. **DSL 结构**：query + sort + from/size + highlight + aggs
2. **叶子查询**：match（分词）/ term（不分词）/ range（范围）/ match_all（全部）
3. **复合查询**：bool（must/filter/should/must_not）、function_score（竞价排名）
4. **must vs filter**：都要满足，但 must 算分 filter 不算分，filter 性能更好
5. **高亮**：match 查询 + highlight 配置 + 解析时替换原始字段
6. **聚合**：桶（分组）+ 度量（计算）+ 管道（二次计算），text 字段不能聚合
7. **深度分页**：from+size > 10000 会报错，实际项目限制页码即可

## 十二、关联笔记

- [[ES入门与IK分词器]]
- [[RestClient操作ES：索引库与文档CRUD]]
