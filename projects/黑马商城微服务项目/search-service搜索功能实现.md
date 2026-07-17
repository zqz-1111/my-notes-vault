---
date: 2026-07-17
tags: [黑马商城微服务项目, Elasticsearch, SpringCloud]
---

# 【Elasticsearch】search-service 搜索功能实现：搜索接口 + 过滤聚合 + 竞价排名

## 一、写在前面

前面学完了 ES 的 DSL 语法和 RestClient API，今天把它们用到真实业务里——在 search-service 中实现三个核心搜索接口：

1. **搜索接口** `/search/list` — 带过滤、排序、分页的商品搜索
2. **过滤聚合接口** `/search/filters` — 根据搜索条件动态返回分类/品牌筛选项
3. **竞价排名** — 用 function_score 让广告商品排在前面

学完这篇你能掌握：ES bool query 在业务中的实际用法、聚合查询实现动态筛选、function_score 算分控制排名。

## 二、前置准备

- search-service 模块已创建，RestHighLevelClient 已配置
- ES 索引库 items 已建好，数据已导入
- Gateway 路由 `/search/** → lb://search-service`

## 三、搜索接口实现

### 3.1 接口信息

- 请求方式：GET
- 请求路径：/search/list
- 参数：key（关键字）、pageNo、pageSize、sortBy、isAsc、category、brand、minPrice、maxPrice

### 3.2 核心代码：bool query 构建

```java
// SearchServiceImpl.java
private BoolQueryBuilder buildQuery(ItemPageQuery query) {
    BoolQueryBuilder boolQuery = QueryBuilders.boolQuery()
            .must(StrUtil.isNotBlank(key)
                    ? QueryBuilders.matchQuery("name", key)
                    : QueryBuilders.matchAllQuery())
            .filter(QueryBuilders.termQuery("status", 1));

    // 不满足条件就不添加 filter，不要用 matchAllQuery() 作 filter 兜底！
    if (StrUtil.isNotBlank(brand)) {
        boolQuery.filter(QueryBuilders.termQuery("brand", brand));
    }
    if (StrUtil.isNotBlank(category)) {
        boolQuery.filter(QueryBuilders.termQuery("category", category));
    }
    if (minPrice != null || maxPrice != null) {
        boolQuery.filter(QueryBuilders.rangeQuery("price").gte(minPrice).lte(maxPrice));
    }
    return boolQuery;
}
```

关键点：
- `must` 放关键字搜索（会算分）
- `filter` 放精确过滤（不算分，性能更好）
- **不要用 `matchAllQuery()` 作 filter 的兜底值**，会导致 ES 行为异常，搜"电脑"出来全是拉杆箱

### 3.3 动态排序

```java
if (StrUtil.isNotBlank(sortBy)) {
    SortOrder order = Boolean.TRUE.equals(isAsc) ? SortOrder.ASC : SortOrder.DESC;
    request.source().sort(sortBy, order);
} else {
    // 默认按相关性评分排序
    request.source().sort("_score", SortOrder.DESC);
}
```

前端传 `sortBy=price&isAsc=true` 就按价格升序，不传就按默认排序。

### 3.4 Controller 层

```java
@ApiOperation("搜索商品")
@GetMapping("/list")
public PageDTO<ItemDTO> search(ItemPageQuery query) {
    return searchService.search(query);
}
```

参数直接用 `ItemPageQuery` 接收，GET 请求自动绑定。

## 四、过滤条件聚合

### 4.1 业务需求

搜索页面的分类/品牌筛选项应该是动态的——搜"手机"只显示手机相关的分类和品牌，搜"电视"只显示电视相关的。

### 4.2 接口信息

- 请求方式：POST
- 请求路径：/search/filters
- 请求参数：与搜索参数一致（key、category、brand、price 等）
- 返回格式：

```json
{
    "category": ["手机", "曲面电视", "拉杆箱"],
    "brand": ["华为", "小米", "Apple"]
}
```

### 4.3 核心：filter 聚合 + 排除自身过滤

关键思路：聚合某个字段时，要**排除该字段自身的过滤条件**，否则选了"手机"分类，分类聚合结果也只有"手机"，没有意义。

```java
@Override
public Map<String, List<String>> filters(ItemPageQuery query) {
    SearchRequest request = new SearchRequest(INDEX_NAME);

    // 聚合分类：排除 category 过滤，保留 key + brand + status 等
    BoolQueryBuilder categoryQuery = buildQueryWithout(query, "category");
    // 聚合品牌：排除 brand 过滤，保留 key + category + status 等
    BoolQueryBuilder brandQuery = buildQueryWithout(query, "brand");

    request.source()
            .query(QueryBuilders.matchAllQuery())
            .aggregation(AggregationBuilders.filter("category_agg", categoryQuery)
                    .subAggregation(AggregationBuilders.terms("category").field("category").size(50)))
            .aggregation(AggregationBuilders.filter("brand_agg", brandQuery)
                    .subAggregation(AggregationBuilders.terms("brand").field("brand").size(50)))
            .size(0); // 不需要搜索结果，只要聚合

    // 发送请求并解析...
}
```

### 4.4 buildQueryWithout 方法

```java
private BoolQueryBuilder buildQueryWithout(ItemPageQuery query, String excludeField) {
    BoolQueryBuilder boolQuery = QueryBuilders.boolQuery()
            .must(StrUtil.isNotBlank(key) ? QueryBuilders.matchQuery("name", key) : QueryBuilders.matchAllQuery())
            .filter(QueryBuilders.termQuery("status", 1));

    // 只添加非排除字段的过滤条件
    if (!"brand".equals(excludeField) && StrUtil.isNotBlank(brand)) {
        boolQuery.filter(QueryBuilders.termQuery("brand", brand));
    }
    if (!"category".equals(excludeField) && StrUtil.isNotBlank(category)) {
        boolQuery.filter(QueryBuilders.termQuery("category", category));
    }
    // ...
    return boolQuery;
}
```

### 4.5 解析聚合结果

聚合结构是：`Filter → subAggregation → Terms`

```java
// 先取 Filter 聚合
Filter categoryFilter = aggregations.get("category_agg");
// 再从 Filter 的子聚合中取 Terms
Terms categoryTerms = categoryFilter.getAggregations().get("category");
// 遍历桶
for (Terms.Bucket bucket : categoryTerms.getBuckets()) {
    categories.add(bucket.getKeyAsString());
}
```

**注意**：`Terms` 类没有 `getAggregations()` 方法，要从 `Filter` 对象上取子聚合。

## 五、竞价排名（function_score）

### 5.1 需求

数据库中 `isAD=true` 的商品是广告商品，需要排在搜索结果最前面。

### 5.2 原理

ES 默认按 `_score`（相关性评分）排序。`function_score` 查询可以在原有算分基础上，对满足条件的文档加权：

- `isAD=true` 的商品，权重 ×10
- 其他商品权重不变
- 结果：广告商品分数更高，排在前面

### 5.3 代码实现

```java
// 用 function_score 包裹原来的 bool query
return QueryBuilders.functionScoreQuery(
        boolQuery,
        new FunctionScoreQueryBuilder.FilterFunctionBuilder[]{
                new FunctionScoreQueryBuilder.FilterFunctionBuilder(
                        QueryBuilders.termQuery("isAD", true),
                        ScoreFunctionBuilders.weightFactorFunction(10)
                )
        }
).boostMode(CombineFunction.MULTIPLY);
```

### 5.4 排序配合

竞价排名依赖 `_score` 排序才能生效：

```java
// 默认按评分排序（不传 sortBy 时）
request.source().sort("_score", SortOrder.DESC);
```

如果用户指定了排序字段（如价格），就按指定字段排序，`_score` 作为同分时的 tiebreaker。

## 六、踩坑避坑指南

### 报错坑

**坑 1：matchAllQuery() 作 filter 导致搜索结果异常**
- 问题：`.filter(StrUtil.isNotBlank(brand) ? QueryBuilders.termQuery("brand", brand) : QueryBuilders.matchAllQuery())`
- 后果：搜"电脑"出来全是拉杆箱，过滤不生效
- 原因：`matchAllQuery()` 放在 bool query 的 filter 里，ES 会优化处理，导致其他 filter 条件失效
- 正确做法：不满足条件就不添加 filter

**坑 2：Terms 类没有 getAggregations() 方法**
- 报错：`无法解析 'Terms' 中的方法 'getAggregations'`
- 原因：聚合结构是 Filter → Terms，要从 Filter 对象上取子聚合
- 正确写法：`filter.getAggregations().get("terms名称")`

**坑 3：search-service 和 cart-service 端口冲突**
- 两个服务都是 8082，启动失败
- 解决：search-service 改成 8083

### 逻辑坑

**坑 4：前端 isTest 默认开启**
- 前端 `search.html` 有个 `isTest: true` 开关，开启时用假数据不调接口
- 需要改成 `false` 才能调真实 API

**坑 5：主页搜索跳转不带 key 参数**
- `index.html` 的 search() 方法逻辑反了，有关键字时反而跳到不带 key 的 URL
- 修复：`if (!this.key || this.key === "null")` 时跳不带 key 的 URL

## 七、效果验证

- 搜"手机" → 结果只有手机，广告商品排第一
- 搜"电视" → 结果只有电视，过滤栏动态显示电视相关的分类和品牌
- 点品牌筛选 → 结果只显示该品牌
- 点价格排序 → 结果按价格排列

## 八、关联笔记

- [[ES DSL查询语法与RestClient搜索API]]
- [[search-service搜索服务拆分]]

## 九、代码变更

- 修改：`search-service/.../SearchServiceImpl.java` — 搜索接口 + 过滤聚合 + 竞价排名
- 修改：`search-service/.../SearchController.java` — 新增 POST /search/filters 端点
- 修改：`search-service/.../ISearchService.java` — 新增 filters 方法
- 修改：`search-service/.../ItemDoc.java` — 新增 status 字段
- 修改：`item-service/.../ItemDoc.java` — 新增 status 字段（MQ 同步用）
- 修改：`item-service/.../IndexTest.java` — ES mapping 新增 status 字段
- 修改：`search-service/.../application.yaml` — 端口改为 8083
- 修改：`hmall-nginx/.../index.html` — 修复主页搜索跳转逻辑
- 修改：`hmall-nginx/.../search.html` — isTest 改为 false
