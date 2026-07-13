---
date: 2026-07-13
tags:
  - Elasticsearch
  - RestClient
  - 黑马商城
---

# 【Elasticsearch】RestClient 操作 ES：索引库与文档 CRUD

## 一、写在前面

1. 为什么学 RestClient？—— Kibana DevTools 只能手动测试，Java 项目中需要通过代码操作 ES，RestHighLevelClient 是 7.x 版本的标准方案
2. 学完这篇你能掌握：Mapping 映射设计、索引库 CRUD、文档 CRUD、批量导入数据的 Java 代码实现

## 二、前置准备

- ES 7.12.1 已部署（见 [[ES入门与IK分词器]]）
- item-service 模块已创建
- IK 分词器已安装

## 三、整合 RestClient

### 3.1 引入依赖

```xml
<!--elasticsearch-->
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
</dependency>
```

### 3.2 覆盖 ES 版本

SpringBoot 默认带的是 7.17.10，和咱们的 7.12.1 不兼容，必须覆盖：

```xml
<properties>
    <elasticsearch.version>7.12.1</elasticsearch.version>
</properties>
```

**踩坑：** 版本不一致会报各种奇怪错误（类找不到、方法签名不匹配）。

### 3.3 初始化客户端

```java
RestHighLevelClient client = new RestHighLevelClient(
    RestClient.builder(HttpHost.create("http://192.168.188.130:9200"))
);
```

- `RestHighLevelClient` 是**线程安全的**，通常做成单例 Bean
- 记得换成自己虚拟机的 IP

## 四、Mapping 映射设计

### 4.1 设计原则

| 场景 | type | 说明 |
|---|---|---|
| 需要全文搜索 | `text` | 配合分词器（如商品名） |
| 精确匹配/过滤 | `keyword` | 品牌、分类、ID |
| 排序/范围查询 | `integer`/`date` | 价格、销量、时间 |
| 只展示不搜索 | `keyword` + `index: false` | 图片 URL |

### 4.2 黑马商城商品索引库

```json
PUT /items
{
  "mappings": {
    "properties": {
      "id": { "type": "keyword" },
      "name": { "type": "text", "analyzer": "ik_max_word" },
      "price": { "type": "integer" },
      "stock": { "type": "integer" },
      "image": { "type": "keyword", "index": false },
      "category": { "type": "keyword" },
      "brand": { "type": "keyword" },
      "sold": { "type": "integer" },
      "commentCount": { "type": "integer", "index": false },
      "isAD": { "type": "boolean" },
      "updateTime": { "type": "date" }
    }
  }
}
```

**设计要点：**
- `name` 用 `text` + `ik_max_word`：商品名要全文搜索，细粒度分词建更多词条
- `price` 用 `integer`：价格单位是**分**，不是元，避免浮点精度问题
- `image` 和 `commentCount` 加 `index: false`：只展示不搜索，省空间

## 五、索引库操作（Kibana DevTools）

### 5.1 CRUD 速查

| 操作 | 方式 | 路径 | 请求体 |
|---|---|---|---|
| 创建 | `PUT` | `/索引库名` | mappings |
| 查询 | `GET` | `/索引库名` | 无 |
| 修改 | `PUT` | `/索引库名/_mapping` | 只能加新字段 |
| 删除 | `DELETE` | `/索引库名` | 无 |

### 5.2 关键规则

- **已有字段不能改**，只能新增字段（改了倒排索引要重建）
- **删除索引库不可逆**，数据全没
- `index: false` 的字段写入能存，但搜不到

## 六、文档操作（Kibana DevTools）

### 6.1 CRUD 速查

| 操作 | 方式 | 路径 | 请求体 |
|---|---|---|---|
| 新增 | `POST` | `/_doc/{id}` | 完整 JSON |
| 查询 | `GET` | `/_doc/{id}` | 无 |
| 删除 | `DELETE` | `/_doc/{id}` | 无 |
| 全量修改 | `PUT` | `/_doc/{id}` | 完整 JSON |
| 局部修改 | `POST` | `/_update/{id}` | `{"doc":{}}` |
| 批处理 | `POST` | `/_bulk` | action+数据行交替 |

### 6.2 关键规则

- 全量修改本质是**先删后增**，ID 不存在会变成新增
- 局部修改必须包一层 `doc`
- 批处理每行一个 JSON，action 和数据行交替，delete 不需要数据行

## 七、RestClient 操作索引库

### 7.1 套路统一

```
创建 Request → 准备参数 → 发送请求
```

### 7.2 创建索引库

```java
@Test
void testCreateIndex() throws IOException {
    // 1.创建Request对象
    CreateIndexRequest request = new CreateIndexRequest("items");
    // 2.准备请求参数
    request.source(MAPPING_TEMPLATE, XContentType.JSON);
    // 3.发送请求
    client.indices().create(request, RequestOptions.DEFAULT);
}
```

`MAPPING_TEMPLATE` 是 JSON 字符串常量，存放 Mapping 定义。

### 7.3 删除索引库

```java
@Test
void testDeleteIndex() throws IOException {
    // 1.创建Request对象
    DeleteIndexRequest request = new DeleteIndexRequest("items");
    // 2.发送请求
    client.indices().delete(request, RequestOptions.DEFAULT);
}
```

### 7.4 判断索引库是否存在

```java
@Test
void testExistsIndex() throws IOException {
    // 1.创建Request对象
    GetIndexRequest request = new GetIndexRequest("items");
    // 2.发送请求
    boolean exists = client.indices().exists(request, RequestOptions.DEFAULT);
    // 3.输出
    System.err.println(exists ? "索引库已经存在！" : "索引库不存在！");
}
```

### 7.5 索引库操作速查

| 操作 | Request 类 | 方法 |
|---|---|---|
| 创建 | `CreateIndexRequest` | `client.indices().create()` |
| 删除 | `DeleteIndexRequest` | `client.indices().delete()` |
| 判断存在 | `GetIndexRequest` | `client.indices().exists()` |

**踩坑：** `DeleteIndexRequest` 的包路径是 `org.elasticsearch.action.admin.indices.delete`，不是 `org.elasticsearch.client.indices`，和 `CreateIndexRequest` 不一样！

## 八、RestClient 操作文档

### 8.1 套路统一

```
创建 Request → 准备参数 → 发送请求 → 解析结果（查询时）
```

### 8.2 索引库 vs 文档操作对比

| | 索引库操作 | 文档操作 |
|---|---|---|
| 请求方式 | `client.indices().xxx()` | `client.xxx()` |

### 8.3 新增文档

```java
@Test
void testAddDocument() throws IOException {
    // 1.根据id查询商品数据
    Item item = itemService.getById(100002644680L);
    // 2.转换为文档类型
    ItemDoc itemDoc = BeanUtil.copyProperties(item, ItemDoc.class);
    // 3.将ItemDoc转json
    String doc = JSONUtil.toJsonStr(itemDoc);

    // 1.准备Request对象
    IndexRequest request = new IndexRequest("items").id(itemDoc.getId());
    // 2.准备Json文档
    request.source(doc, XContentType.JSON);
    // 3.发送请求
    client.index(request, RequestOptions.DEFAULT);
}
```

**关键点：**
- 数据库查的是 `Item` 对象，ES 存的是 `ItemDoc` 对象
- 用 `BeanUtil.copyProperties` 转换，用 `JSONUtil.toJsonStr` 序列化
- `IndexRequest` 的 `id()` 指定文档 ID

### 8.4 查询文档

```java
@Test
void testGetDocumentById() throws IOException {
    // 1.准备Request对象
    GetRequest request = new GetRequest("items").id("100002644680");
    // 2.发送请求
    GetResponse response = client.get(request, RequestOptions.DEFAULT);
    // 3.获取响应结果中的source
    String json = response.getSourceAsString();

    ItemDoc itemDoc = JSONUtil.toBean(json, ItemDoc.class);
    System.out.println("itemDoc= " + itemDoc);
}
```

**关键点：** 实际数据在 `_source` 里，用 `getSourceAsString()` 拿到 JSON 字符串。

### 8.5 删除文档

```java
@Test
void testDeleteDocument() throws IOException {
    // 1.准备Request，两个参数：索引库名、文档id
    DeleteRequest request = new DeleteRequest("items", "100002644680");
    // 2.发送请求
    client.delete(request, RequestOptions.DEFAULT);
}
```

### 8.6 局部修改文档

```java
@Test
void testUpdateDocument() throws IOException {
    // 1.准备Request
    UpdateRequest request = new UpdateRequest("items", "100002644680");
    // 2.准备请求参数
    request.doc(
        "price", 58800,
        "commentCount", 1
    );
    // 3.发送请求
    client.update(request, RequestOptions.DEFAULT);
}
```

### 8.7 批量导入

```java
@Test
void testLoadItemDocs() throws IOException {
    // 分页查询商品数据
    int pageNo = 1;
    int size = 1000;
    while (true) {
        Page<Item> page = itemService.lambdaQuery()
            .eq(Item::getStatus, 1)
            .page(new Page<>(pageNo, size));
        // 非空校验
        List<Item> items = page.getRecords();
        if (CollUtil.isEmpty(items)) {
            return;
        }
        log.info("加载第{}页数据，共{}条", pageNo, items.size());
        // 1.创建Request
        BulkRequest request = new BulkRequest("items");
        // 2.准备参数，添加多个新增的Request
        for (Item item : items) {
            ItemDoc itemDoc = BeanUtil.copyProperties(item, ItemDoc.class);
            request.add(new IndexRequest()
                .id(itemDoc.getId())
                .source(JSONUtil.toJsonStr(itemDoc), XContentType.JSON));
        }
        // 3.发送请求
        client.bulk(request, RequestOptions.DEFAULT);
        // 翻页
        pageNo++;
    }
}
```

**关键点：**
- 每次查 1000 条，避免内存溢出
- 只查上架商品（`status=1`）
- `BulkRequest` 的 `add()` 方法可以添加 `IndexRequest`、`UpdateRequest`、`DeleteRequest`

### 8.8 文档操作速查

| 操作 | Request 类 | 方法 |
|---|---|---|
| 新增/全量修改 | `IndexRequest` | `client.index()` |
| 查询 | `GetRequest` | `client.get()` |
| 删除 | `DeleteRequest` | `client.delete()` |
| 局部修改 | `UpdateRequest` | `client.update()` |
| 批量操作 | `BulkRequest` | `client.bulk()` |

## 九、ItemDoc 实体类

索引库结构与数据库结构有差异，需要定义一个对应的实体：

```java
@Data
@ApiModel(description = "索引库实体")
public class ItemDoc {
    @ApiModelProperty("商品id")
    private String id;           // ES用String，数据库用Long
    @ApiModelProperty("商品名称")
    private String name;
    @ApiModelProperty("价格（分）")
    private Integer price;
    @ApiModelProperty("商品图片")
    private String image;
    @ApiModelProperty("类目名称")
    private String category;
    @ApiModelProperty("品牌名称")
    private String brand;
    @ApiModelProperty("销量")
    private Integer sold;
    @ApiModelProperty("评论数")
    private Integer commentCount;
    @ApiModelProperty("是否是推广广告，true/false")
    private Boolean isAD;        // ES用Boolean，数据库用Integer
    @ApiModelProperty("更新时间")
    private LocalDateTime updateTime;
}
```

**ItemDoc vs Item 的区别：**
- `id` 用 `String`：ES 的 keyword 类型对应 String
- 去掉 `stock`/`status`：搜索页面不展示
- `isAD` 转 `Boolean`：ES 里存布尔值更直观

## 十、踩坑避坑指南

### 逻辑坑

**坑 1：DeleteIndexRequest 包路径不同**
- 问题：`CreateIndexRequest` 在 `org.elasticsearch.client.indices`，但 `DeleteIndexRequest` 在 `org.elasticsearch.action.admin.indices.delete`
- 后果：导入错误的包，编译报错
- 正确做法：注意区分，`GetIndexRequest` 和 `CreateIndexRequest` 同包，`DeleteIndexRequest` 单独一个包

**坑 2：全量修改 ID 不存在会变成新增**
- 问题：用 `IndexRequest` 做全量修改时，如果 ID 不存在，不会报错，而是新增
- 后果：可能误操作
- 正确做法：确认 ID 存在再做全量修改，或用局部修改 `UpdateRequest`

### 报错坑

**坑 3：ES 版本不一致**
- 报错信息：类找不到、方法签名不匹配
- 原因分析：SpringBoot 默认 ES 版本 7.17.10，项目用 7.12.1
- 解决方案：`pom.xml` 中加 `<elasticsearch.version>7.12.1</elasticsearch.version>` 覆盖

## 十一、全文总结

1. **RestClient 套路统一**：创建 Request → 准备参数 → 发送请求
2. **索引库操作**用 `client.indices().xxx()`，文档操作用 `client.xxx()`
3. **Mapping 设计**要根据业务需求决定字段类型，text 用于搜索，keyword 用于精确匹配
4. **批量导入**用 `BulkRequest`，分页查询避免内存溢出
5. **ItemDoc** 是 ES 的文档实体，和数据库的 Item 有差异（id 类型、字段取舍）

## 十二、关联笔记

- [[ES入门与IK分词器]]
- [[../09-RabbitMQ/RabbitMQ入门|RabbitMQ入门]]

## 十三、代码变更

- 新增：`item-service/pom.xml` — 添加 ES 依赖 + 覆盖版本
- 新增：`item-service/src/main/java/com/hmall/item/domain/po/ItemDoc.java` — ES 索引库实体
- 新增：`item-service/src/test/java/com/hmall/item/es/IndexTest.java` — 索引库 CRUD 测试
- 新增：`item-service/src/test/java/com/hmall/item/es/DocumentTest.java` — 文档 CRUD 测试
