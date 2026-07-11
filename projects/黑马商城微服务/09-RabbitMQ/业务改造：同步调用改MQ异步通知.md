---
date: 2026-07-11
tags: [RabbitMQ, 黑马商城, 异步改造, SpringAMQP]
---

# 业务改造：同步调用改MQ异步通知

## 一、写在前面

微服务之间用 OpenFeign 同步调用，一旦下游服务挂了，上游也跟着崩。把非核心链路改成 MQ 异步通知，能实现**服务解耦 + 故障隔离**。

学完这篇你能掌握：
- 余额支付成功后，用 MQ 异步通知交易服务更新订单状态
- 下单成功后，用 MQ 异步通知购物车服务清理商品
- MQ 配置抽取到 Nacos 共享配置

## 二、改造一：支付成功异步通知

### 2.1 原来的同步调用链路

```
pay-service  --Feign-->  trade-service (markOrderPaySuccess)
```

问题：trade-service 挂了，支付也失败。

### 2.2 改造后的异步链路

```
pay-service  --MQ-->  pay.direct (exchange)  -->  trade.pay.success.queue  -->  trade-service
```

### 2.3 pay-service（生产者）

**pom.xml 加依赖：**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

**bootstrap.yaml 加 MQ 连接配置：**
```yaml
spring:
  rabbitmq:
    host: 192.168.188.130
    port: 5672
    virtual-host: /hmall
    username: hmall
    password: 123
```

**PayOrderServiceImpl 改造：**
```java
// 改造前
tradeClient.markOrderPaySuccess(po.getBizOrderNo());

// 改造后
rabbitTemplate.convertAndSend("pay.direct", "pay.success", po.getBizOrderNo());
```

注解从 `@GlobalTransactional` 改成 `@Transactional`（MQ 异步不需要 Seata 跨服务事务）。

### 2.4 trade-service（消费者）

**PayStatusListener.java：**
```java
@Component
@RequiredArgsConstructor
public class PayStatusListener {

    private final IOrderService orderService;

    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "trade.pay.success.queue", durable = "true"),
            exchange = @Exchange(name = "pay.direct"),
            key = "pay.success"
    ))
    public void listenPaySuccess(Long orderId){
        orderService.markOrderPaySuccess(orderId);
    }
}
```

`@RabbitListener` 注解式声明：队列 + 交换机 + 绑定关系一步搞定。

## 三、改造二：下单成功异步清理购物车

### 3.1 原来的同步调用

```
trade-service  --Feign-->  cart-service (deleteCartItemByIds)
```

### 3.2 改造后的异步链路

```
trade-service  --MQ-->  trade.topic (exchange)  -->  cart.clear.queue  -->  cart-service
```

### 3.3 定义消息体 OrderMessage

```java
@Data
public class OrderMessage implements Serializable {
    private static final long serialVersionUID = 1L;
    private List<Long> itemIds;
}
```

> **踩坑：**必须实现 `Serializable` + 加 `serialVersionUID`，否则 RabbitMQ 默认的 `SimpleMessageConverter` 无法序列化。

### 3.4 trade-service（生产者）

```java
OrderMessage orderMessage = new OrderMessage();
orderMessage.setItemIds(new ArrayList<>(itemIds));
// userId 由 MqConfig 自动写入消息 Header，无需手动设置
rabbitTemplate.convertAndSend("trade.topic", "order.create", orderMessage);
```

### 3.5 cart-service（消费者）

```java
@Component
@RequiredArgsConstructor
public class ClearCartListener {

    private final ICartService cartService;

    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "cart.clear.queue", durable = "true"),
            exchange = @Exchange(name = "trade.topic", type = "topic"),
            key = "order.create"
    ))
    public void listenClearCart(OrderMessage message) {
        // UserContext 由 MqConfig 自动设置
        cartService.lambdaUpdate()
                .eq(Cart::getUserId, UserContext.getUser())
                .in(Cart::getItemId, message.getItemIds())
                .remove();
    }
}
```

## 四、MQ 共享配置抽取到 Nacos

跟 `shared-jdbc.yaml`、`shared-seata.yaml` 一个套路：

1. Nacos 创建 `shared-mq.yaml`，内容是 `spring.rabbitmq` 连接信息
2. 各微服务 `bootstrap.yaml` 引入：
```yaml
shared-configs:
  - dataId: shared-mq.yaml
```
3. 删掉各服务本地的 `spring.rabbitmq` 配置块

好处：MQ 地址变了只改 Nacos 一处，所有服务自动生效。

## 五、方案对比：direct vs topic 交换机

| 特性 | direct | topic |
|---|---|---|
| 路由规则 | 精确匹配 RoutingKey | 通配符匹配（`#` 多词、`*` 一词） |
| 适用场景 | 点对点通知（支付→更新订单） | 一对多广播（下单→清理购物车/发短信/加积分） |
| 灵活性 | 低 | 高 |

本次改造：
- 支付通知用 `pay.direct`（精确匹配 `pay.success`）
- 下单通知用 `trade.topic`（通配符 `order.create`，以后扩展方便）

## 六、踩坑避坑指南

### 坑 1：SimpleMessageConverter 序列化失败

- 报错：`SimpleMessageConverter only supports String, byte[] and Serializable payloads`
- 原因：`OrderMessage` 没实现 `Serializable`
- 解决：加 `implements Serializable` + `private static final long serialVersionUID = 1L`

### 坑 2：反序列化 InvalidClassException

- 报错：`InvalidClassException: class invalid for deserialization`
- 原因：加 `serialVersionUID` 之前发的消息还卡在队列里，旧消息的 class UID 不匹配
- 解决：去 RabbitMQ 管理页面 Purge 掉旧消息，重新下单

### 坑 3：RabbitTemplateCustomizer 找不到

- 报错：`无法解析符号 RabbitTemplateCustomizer`
- 原因：`RabbitTemplateCustomizer` 在 `spring-boot-autoconfigure` 中，hm-common 模块没有直接依赖
- 解决：改用 `BeanPostProcessor` 自定义 `RabbitTemplate`

### 坑 4：Advice 类型不兼容

- 报错：`不兼容的类型，实际为 UserContextAdvice，需要 Advice`
- 原因：`AbstractPointcutAdvisor` 是 `Advisor`，不是 `Advice`
- 解决：直接用 `MethodInterceptor`（它继承自 `Advice`）

## 七、全文总结

- 同步改异步的核心：Feign 调用 → `rabbitTemplate.convertAndSend()` + `@RabbitListener` 消费
- MQ 配置统一管理：`shared-mq.yaml` 抽到 Nacos，跟其他共享配置一个套路
- 消息体必须实现 `Serializable` + `serialVersionUID`
- 交换机选型：点对点用 `direct`，一对多用 `topic`

## 八、关联笔记

- [[初识MQ：同步调用问题与异步架构]]
- [[交换机实战、声明方式与消息转换器]]
- [[SpringAMQP入门与整合]]

## 九、代码变更

- 修改：`pay-service/pom.xml` — 加 spring-boot-starter-amqp 依赖
- 修改：`trade-service/pom.xml` — 加 spring-boot-starter-amqp 依赖
- 修改：`cart-service/pom.xml` — 加 spring-boot-starter-amqp 依赖
- 修改：`pay-service/bootstrap.yaml` — 加 shared-mq.yaml 共享配置
- 修改：`trade-service/bootstrap.yaml` — 加 shared-mq.yaml 共享配置
- 修改：`cart-service/bootstrap.yaml` — 加 shared-mq.yaml 共享配置
- 修改：`pay-service/PayOrderServiceImpl.java` — Feign 调用改 MQ 发消息
- 修改：`trade-service/OrderServiceImpl.java` — Feign 调用改 MQ 发消息
- 新增：`trade-service/PayStatusListener.java` — 监听支付成功消息
- 新增：`cart-service/ClearCartListener.java` — 监听下单消息清理购物车
- 新增：`hm-api/OrderMessage.java` — 下单消息 DTO
