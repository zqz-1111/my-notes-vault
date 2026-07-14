---
date: 2026-07-14
tags: [hmall, 微服务, Elasticsearch, 服务拆分]
---

# search-service 搜索服务拆分

## 一、写在前面

搜索业务并发压力高，跟商品服务混在一起不方便独立优化。把搜索功能从 item-service 抽出来，单独建一个 search-service 微服务，后端从 MySQL LIKE 查询改成 Elasticsearch 查询。

学完这篇你能掌握：
- 如何从已有微服务中拆出一个新服务
- ES RestHighLevelClient 在 Spring Boot 中的生产级用法
- Gateway 路由配置

## 二、新建 search-service 模块

### 2.1 目录结构

```
search-service/
├── pom.xml
└── src/main/
    ├── java/com/hmall/search/
    │   ├── SearchApplication.java
    │   ├── config/ElasticSearchConfig.java
    │   ├── controller/SearchController.java
    │   ├── domain/
    │   │   ├── dto/ItemDTO.java
    │   │   ├── po/ItemDoc.java
    │   │   └── query/ItemPageQuery.java
    │   └── service/
    │       ├── ISearchService.java
    │       └── impl/SearchServiceImpl.java
    └── resources/
        ├── application.yaml
        └── bootstrap.yaml
```

### 2.2 pom.xml 关键依赖

```xml
<!-- 不需要 MySQL、MyBatis、Seata -->
<dependency>
    <groupId>com.heima</groupId>
    <artifactId>hm-common</artifactId>
</dependency>
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
</dependency>
<!-- PageQuery 继承需要 MyBatis-Plus 的 Page 类 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-extension</artifactId>
    <scope>provided</scope>
</dependency>
```

### 2.3 配置文件

**application.yaml**：
```yaml
server:
  port: 8082
hm:
  swagger:
    title: 搜索服务接口文档
    package: com.hmall.search.controller
  es:
    host: 192.168.188.130
    port: 9200
```

**bootstrap.yaml**：
```yaml
spring:
  application:
    name: search-service
  profiles:
    active: dev
  cloud:
    nacos:
      server-addr: 192.168.188.130:8848
      config:
        file-extension: yaml
        shared-configs:
          - dataId: shared-log.yaml
          - dataId: shared-swagger.yaml
          - dataId: shared-mq.yaml  # 后面做数据同步时加的
```

## 三、ES 配置类

```java
@Slf4j
@Data
@Configuration
@ConfigurationProperties(prefix = "hm.es")
public class ElasticSearchConfig {

    private String host;
    private Integer port;

    @Bean
    public RestHighLevelClient restHighLevelClient() {
        return new RestHighLevelClient(RestClient.builder(
                HttpHost.create("http://" + host + ":" + port)
        ));
    }
}
```

用 `@ConfigurationProperties` 从配置文件读取 host/port，不用硬编码。

## 四、搜索核心逻辑

把原来 MySQL 的 `lambdaQuery` 翻译成 ES 的 `bool query`：

| 原来（MySQL） | 现在（ES） |
|---|---|
| `like name` | `match name`（ik_smart 分词） |
| `eq brand` | `term brand`（精确匹配） |
| `eq category` | `term category`（精确匹配） |
| `between price` | `range price gte/lte` |
| `order by update_time desc` | `sort updateTime DESC` |
| `page(pageNo, pageSize)` | `from + size` |

```java
@Override
public PageDTO<ItemDTO> search(ItemPageQuery query) {
    SearchRequest request = new SearchRequest(INDEX_NAME);
    request.source()
            .query(QueryBuilders.boolQuery()
                    .must(StrUtil.isNotBlank(key)
                            ? QueryBuilders.matchQuery("name", key)
                            : QueryBuilders.matchAllQuery())
                    .filter(StrUtil.isNotBlank(brand)
                            ? QueryBuilders.termQuery("brand", brand)
                            : QueryBuilders.matchAllQuery())
                    .filter(StrUtil.isNotBlank(category)
                            ? QueryBuilders.termQuery("category", category)
                            : QueryBuilders.matchAllQuery())
                    .filter(minPrice != null || maxPrice != null
                            ? QueryBuilders.rangeQuery("price").gte(minPrice).lte(maxPrice)
                            : QueryBuilders.matchAllQuery())
            )
            .sort("updateTime", SortOrder.DESC)
            .from((pageNo - 1) * pageSize)
            .size(pageSize);
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 解析结果...
}
```

## 五、Gateway 路由

在 `hm-gateway/application.yaml` 加路由：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: search-service
          uri: lb://search-service
          predicates:
            - Path=/search/**
```

`/search/**` 已经在白名单里（免登录），不用额外配置。

## 六、踩坑避坑

### 坑 1：search-service 编译报找不到 getter

- 原因：Lombok 注解处理器没生效（终端 Maven 用了 Java 25，Lombok 1.18.20 不支持）
- 解决：在父 pom 的 `maven-compiler-plugin` 显式配置 `annotationProcessorPaths`
- 详见 → [[Lombok与Java版本踩坑指南]]

### 坑 2：缺少 MyBatis-Plus 的 Page 类

- 报错：`找不到 com.baomidou.mybatisplus.extension.plugins.pagination.Page 的类文件`
- 原因：`ItemPageQuery` 继承了 `PageQuery`，`PageQuery` 的 `toMpPage()` 返回 `Page` 类型
- 解决：search-service 加 `mybatis-plus-extension` 依赖（provided scope）

## 七、代码变更

- 新增：`search-service/` 整个模块
- 修改：`pom.xml` — modules 加 search-service
- 修改：`hm-gateway/application.yaml` — 加路由规则
- 删除：`item-service/SearchController.java` — 搜索职责移交
