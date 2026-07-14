---
date: 2026-07-14
tags: [hmall, RabbitMQ, Elasticsearch, 数据同步]
---

# MQ 数据同步：商品变更通知 ES 更新

## 一、写在前面

search-service 的搜索走 ES，但商品数据在 MySQL 里（item-service 管理）。两边数据要保持一致，用 MQ 异步通知实现。

学完这篇你能掌握：
- 生产者发消息 + 消费者监听消息的完整流程
- MQ 消息体选择（对象 vs JSON 字符串）
- @RabbitListener + @QueueBinding 声明式队列

## 二、消息拓扑

| 交换机 | 类型 | 路由键 | 队列 | 生产者 | 消费者 | 消息体 |
|---|---|---|---|---|---|---|
| `item.direct` | direct | `item.es.save` | `item.es.save.queue` | item-service | search-service | `String`（JSON） |
| `item.direct` | direct | `item.es.delete` | `item.es.delete.queue` | item-service | search-service | `Long`（商品ID） |

## 三、生产者（item-service）

### 3.1 pom.xml 加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 3.2 bootstrap.yaml 加共享配置

```yaml
shared-configs:
  - dataId: shared-mq.yaml  # RabbitMQ 连接信息
```

### 3.3 ItemController 发消息

注入 `RabbitMqHelper`，在增删改操作后发消息：

```java
private final RabbitMqHelper rabbitMqHelper;

// 新增/修改 → 发 JSON 字符串
rabbitMqHelper.sendMessage("item.direct", "item.es.save",
        JSONUtil.toJsonStr(BeanUtil.copyProperties(item, ItemDoc.class)));

// 删除 → 发商品ID
rabbitMqHelper.sendMessage("item.direct", "item.es.delete", id);
```

## 四、消费者（search-service）

### 4.1 pom.xml 加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 4.2 bootstrap.yaml 加共享配置

```yaml
shared-configs:
  - dataId: shared-mq.yaml
```

### 4.3 ItemESListener 监听器

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class ItemESListener {

    private final RestHighLevelClient client;

    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "item.es.save.queue", durable = "true"),
            exchange = @Exchange(name = "item.direct"),
            key = "item.es.save"
    ))
    public void listenItemSave(String itemDocJson) {
        ItemDoc itemDoc = JSONUtil.toBean(itemDocJson, ItemDoc.class);
        IndexRequest request = new IndexRequest("items")
                .id(itemDoc.getId())
                .source(itemDocJson, XContentType.JSON);
        client.index(request, RequestOptions.DEFAULT);
    }

    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "item.es.delete.queue", durable = "true"),
            exchange = @Exchange(name = "item.direct"),
            key = "item.es.delete"
    ))
    public void listenItemDelete(Long itemId) {
        DeleteRequest request = new DeleteRequest("items", itemId.toString());
        client.delete(request, RequestOptions.DEFAULT);
    }
}
```

## 五、踩坑避坑

### 坑 1：ItemDoc 对象不能直接传 MQ

- 报错：`SimpleMessageConverter only supports String, byte[] and Serializable payloads`
- 原因：`ItemDoc` 没实现 `Serializable`
- 解决：加了 `implements Serializable` + `serialVersionUID` → 又报 `IOException`
- 根因：`LocalDateTime` 字段在 Java 序列化/反序列化时有兼容性问题
- **最终方案：改成传 JSON 字符串**，用 `JSONUtil.toJsonStr()` 序列化，消费端用 `JSONUtil.toBean()` 反序列化

### 坑 2：item-service 也配了 shared-mq.yaml

- 现象：item-service 连接 RabbitMQ 报 `Connection refused: localhost:5672`
- 原因：item-service 的 bootstrap.yaml 没有引入 `shared-mq.yaml`，RabbitMQ 连接信息是从 Nacos 共享配置读的
- 解决：加上 `- dataId: shared-mq.yaml`

### 坑 3：IOException 编译错误

- 报错：`未报告的异常错误java.io.IOException; 必须对其进行捕获或声明以便抛出`
- 原因：`client.index()` 抛 checked exception，`catch (Exception e) { throw e; }` 编译器不允许
- 解决：`throw new RuntimeException("ES同步写入失败", e);`

## 六、核心经验

**MQ 消息体尽量用 String/JSON，别传复杂对象。** 原因：
1. Java 序列化有各种坑（Serializable、serialVersionUID、字段兼容性）
2. JSON 跨语言、跨版本兼容性好
3. 调试方便（日志直接看到内容）

## 七、代码变更

- 修改：`item-service/pom.xml` — 加 spring-boot-starter-amqp
- 修改：`item-service/bootstrap.yaml` — 加 shared-mq.yaml
- 修改：`item-service/ItemController.java` — 注入 RabbitMqHelper，增删改后发消息
- 修改：`item-service/ItemDoc.java` — 加 Serializable（虽然最后没用对象传）
- 修改：`search-service/pom.xml` — 加 spring-boot-starter-amqp
- 修改：`search-service/bootstrap.yaml` — 加 shared-mq.yaml
- 新增：`search-service/ItemESListener.java` — MQ 监听器，写入/删除 ES

## 八、关联笔记

- [[search-service搜索服务拆分]]
- [[MQ工具抽取：RabbitMqHelper与消费失败处理]]
- [[业务改造：同步调用改MQ异步通知]]
