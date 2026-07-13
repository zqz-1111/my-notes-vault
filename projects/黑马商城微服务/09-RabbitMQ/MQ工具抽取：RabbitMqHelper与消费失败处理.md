---
date: 2026-07-13
tags: [RabbitMQ, 消息队列, 黑马商城, 工具封装]
---

# MQ 工具抽取：RabbitMqHelper 与消费失败处理

## 一、写在前面

1. 为什么学这个？—— MQ 的收发消息、消息可靠性、生产者确认、消费者确认、延迟消息等编码比较复杂，需要封装成工具方便复用
2. 学完你能掌握什么 —— RabbitMqHelper 工具类、消费失败自动投递 error 队列、Nacos 共享 MQ 配置

---

## 二、整体改造内容

| 改造项 | 说明 |
|---|---|
| Nacos 共享配置 | RabbitMQ 连接 + retry 配置统一管理 |
| RabbitMqHelper | 封装普通消息、延迟消息、带确认的消息 |
| MqConsumeErrorAutoConfiguration | 消费失败自动投递 error 队列 |
| 业务代码重构 | 现有代码改用 RabbitMqHelper |

---

## 三、Nacos 共享配置

### shared-mq.yaml

```yaml
spring:
  rabbitmq:
    host: 192.168.188.130
    port: 5672
    virtual-host: /hmall
    username: hmall
    password: 123
    listener:
      simple:
        retry:
          enabled: true          # 开启消费重试
          initial-interval: 1000 # 初始重试间隔 1 秒
          multiplier: 1          # 重试间隔倍数
          max-attempts: 3        # 最大重试次数
          stateless: true        # 无状态
```

### 各服务 bootstrap.yaml 引用

```yaml
spring:
  cloud:
    nacos:
      config:
        shared-configs:
          - dataId: shared-mq.yaml
```

**已确认引用的服务**：trade-service、cart-service、pay-service

**不需要引用的服务**：item-service、user-service、hm-gateway（不用 MQ）

### 删掉本地配置

trade-service 的 application.yaml 中删掉 `spring.rabbitmq` 配置，统一走 Nacos。

---

## 四、RabbitMqHelper 工具类

### 位置

`hm-common/src/main/java/com/hmall/common/utils/RabbitMqHelper.java`

### 三个核心方法

```java
@Slf4j
@RequiredArgsConstructor
public class RabbitMqHelper {

    private final RabbitTemplate rabbitTemplate;

    /**
     * 发送普通消息
     */
    public void sendMessage(String exchange, String routingKey, Object msg) {
        rabbitTemplate.convertAndSend(exchange, routingKey, msg);
    }

    /**
     * 发送延迟消息（需要 DelayExchange 插件支持）
     */
    public void sendDelayMessage(String exchange, String routingKey, Object msg, int delay) {
        rabbitTemplate.convertAndSend(exchange, routingKey, msg, message -> {
            message.getMessageProperties().setDelay(delay);
            return message;
        });
    }

    /**
     * 发送消息并等待 Publisher Confirm（同步阻塞 + 重试）
     */
    public void sendMessageWithConfirm(String exchange, String routingKey, Object msg, int maxRetries) {
        int retries = 0;
        while (retries < maxRetries) {
            try {
                CorrelationData correlationData = new CorrelationData();
                rabbitTemplate.convertAndSend(exchange, routingKey, msg, correlationData);
                // 阻塞等待 confirm 结果
                CorrelationData.Confirm confirm = correlationData.getFuture().get();
                if (confirm.isAck()) {
                    log.debug("消息发送成功，exchange={}, routingKey={}", exchange, routingKey);
                    return;
                } else {
                    log.warn("消息被 nack，exchange={}, routingKey={}, 原因={}", exchange, routingKey, confirm.getReason());
                }
            } catch (Exception e) {
                log.error("消息发送异常，exchange={}, routingKey={}, 第{}次重试", exchange, routingKey, retries + 1, e);
            }
            retries++;
        }
        throw new RuntimeException("消息发送失败，已重试" + maxRetries + "次");
    }
}
```

### 注册为 Bean

在 MqConfig 中注册：

```java
@Bean
public RabbitMqHelper rabbitMqHelper(RabbitTemplate rabbitTemplate) {
    return new RabbitMqHelper(rabbitTemplate);
}
```

---

## 五、MqConsumeErrorAutoConfiguration

### 作用

消费失败时，自动把消息投递到 error 交换机，由专门的 error 队列兜底，避免消息丢失。

### 消息流转

```
消费者处理失败 → 重试 3 次仍然失败
    → RepublishMessageRecoverer
    → error.direct 交换机
    → 微服务名.error.queue 队列
    → 人工处理或定时补偿
```

### 代码实现

```java
@Configuration
@ConditionalOnClass(RabbitTemplate.class)
@ConditionalOnProperty(name = "spring.rabbitmq.listener.simple.retry.enabled", havingValue = "true")
public class MqConsumeErrorAutoConfiguration {

    @Value("${spring.application.name}")
    private String applicationName;

    // error 交换机
    @Bean
    public DirectExchange errorExchange() {
        return new DirectExchange("error.direct");
    }

    // error 队列，名为：微服务名.error.queue
    @Bean
    public Queue errorQueue() {
        return new Queue(applicationName + ".error.queue");
    }

    // 绑定：routingKey = 微服务名
    @Bean
    public Binding errorBinding(Queue errorQueue, DirectExchange errorExchange) {
        return BindingBuilder.bind(errorQueue).to(errorExchange).with(applicationName);
    }

    // 消费失败时重新投递
    @Bean
    public RepublishMessageRecoverer republishMessageRecoverer(RabbitTemplate rabbitTemplate) {
        return new RepublishMessageRecoverer(rabbitTemplate, "error.direct");
    }
}
```

### 关键注解说明

| 注解 | 作用 |
|---|---|
| `@ConditionalOnClass(RabbitTemplate.class)` | 只有引入了 RabbitMQ 依赖才生效 |
| `@ConditionalOnProperty(name = "...retry.enabled", havingValue = "true")` | 只有开启了消费重试才生效 |
| `@Value("${spring.application.name}")` | 动态获取微服务名，每个服务的 error 队列名不同 |

### 动态队列名效果

| 微服务 | error 队列名 |
|---|---|
| trade-service | `trade-service.error.queue` |
| cart-service | `cart-service.error.queue` |
| pay-service | `pay-service.error.queue` |

---

## 六、spring.factories 注册

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.hmall.common.config.MyBatisConfig,\
  com.hmall.common.config.MvcConfig,\
  com.hmall.common.config.MqConfig,\
  com.hmall.common.config.MqConsumeErrorAutoConfiguration
```

**注意**：之前 MqConfig 被重复注册了两次，这次顺便修复了。

---

## 七、业务代码重构

### 改造前（直接用 RabbitTemplate）

```java
private final RabbitTemplate rabbitTemplate;

// 普通消息
rabbitTemplate.convertAndSend("trade.topic", "order.create", orderMessage);

// 延迟消息
rabbitTemplate.convertAndSend("trade.delay.direct", "delay.order.query",
    order.getId(), message -> {
        message.getMessageProperties().setDelay(15 * 60 * 1000);
        return message;
    });
```

### 改造后（用 RabbitMqHelper）

```java
private final RabbitMqHelper rabbitMqHelper;

// 普通消息
rabbitMqHelper.sendMessage("trade.topic", "order.create", orderMessage);

// 延迟消息
rabbitMqHelper.sendDelayMessage("trade.delay.direct", "delay.order.query",
    order.getId(), 15 * 60 * 1000);
```

**改造的文件**：
- `OrderServiceImpl` — 2 处 convertAndSend → sendMessage + sendDelayMessage
- `PayOrderServiceImpl` — 1 处 convertAndSend → sendMessage

---

## 八、消息可靠性总结

| 环节 | 机制 | 实现方式 |
|---|---|---|
| 生产者 → Broker | Publisher Confirm | RabbitMqHelper.sendMessageWithConfirm |
| Broker 存储 | 交换机/队列/消息持久化 | SpringAMQP 默认已持久化 |
| Broker → 消费者 | Consumer ACK | spring.rabbitmq.listener.simple.retry |
| 消费失败 | 重试 + error 队列 | MqConsumeErrorAutoConfiguration |
| 延迟消息 | DelayExchange 插件 | RabbitMqHelper.sendDelayMessage |

---

## 九、踩坑记录

### 坑 1：ListenableFutureCallback 未使用

RabbitMqHelper 里 import 了 `ListenableFutureCallback` 但没用到，编译有警告。

解决：删掉未使用的 import。

### 坑 2：spring.factories 重复注册

原来 MqConfig 被注册了两次。

解决：去掉重复，加上新的 MqConsumeErrorAutoConfiguration。

---

## 十、代码变更

- 新增：`hm-common/.../utils/RabbitMqHelper.java` — MQ 工具类
- 新增：`hm-common/.../config/MqConsumeErrorAutoConfiguration.java` — 消费失败自动配置
- 修改：`hm-common/.../config/MqConfig.java` — 注册 RabbitMqHelper Bean
- 修改：`hm-common/.../spring.factories` — 注册新配置类，去掉重复
- 修改：`trade-service/.../service/impl/OrderServiceImpl.java` — 改用 RabbitMqHelper
- 修改：`pay-service/.../service/impl/PayOrderServiceImpl.java` — 改用 RabbitMqHelper
- 修改：`trade-service/application.yaml` — 删掉本地 RabbitMQ 配置
- 修改：Nacos `shared-mq.yaml` — 加 retry 配置

---

## 十一、关联笔记

- [[延迟消息实战：订单超时自动取消]] — 延迟消息完整业务流程
- [[业务改造：同步调用改MQ异步通知]] — MQ 共享配置抽取 Nacos
- [[登录信息传递优化：MQ消息自动携带用户信息]] — MqConfig 自动装配原理
