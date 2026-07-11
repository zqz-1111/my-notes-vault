---
date: 2026-07-11
tags: [mq-demo, RabbitMQ, 消息队列, SpringAMQP]
---

# RabbitMQ高级篇：消息可靠性与延迟消息

## 一、写在前面

1. 为什么学这个？—— 消息从生产者到消费者，每一步都可能丢失，MQ 高级篇就是教你怎么保证消息不丢
2. 学完你能掌握什么 —— 生产者确认、消费者确认、失败重试、死信队列、业务幂等、延迟消息，全链路可靠性保障

### 业务场景

支付成功后 MQ 通知交易服务更新订单状态。如果消息丢了：
- 支付流水显示"支付成功"
- 订单状态显示"未支付"
- 用户付了钱，订单显示没付 → 用户体验炸裂

**核心原则**：消息应该至少被消费者处理 1 次

---

## 二、生产者可靠性

消息从生产者发出，到进入 Broker，有 4 种丢失场景：

| 场景 | 原因 |
|---|---|
| 连接 RabbitMQ 失败 | MQ 没启动 / 网络断了 |
| Exchange 不存在 | 交换机名写错了 |
| 路由不到 Queue | 路由键不匹配 |
| Broker 内部异常 | MQ 进程出错 |

### 2.1 发送者重试

解决"连不上 MQ"的问题。Spring AMQP 提供消息发送时的重试机制。

```yaml
spring:
  rabbitmq:
    connection-timeout: 1s # 设置MQ的连接超时时间
    template:
      retry:
        enabled: true # 开启超时重试机制
        initial-interval: 1000ms # 失败后的初始等待时间
        multiplier: 1 # 失败后下次的等待时长倍数
        max-attempts: 3 # 最大重试次数
```

**重试间隔计算**：`multiplier=1` 意味着固定间隔 1 秒（1s → 1s → 1s），3 次都失败才抛异常。

#### 踩坑：阻塞式重试

SpringAMQP 的重试是**阻塞式**的，多次重试等待过程中，当前线程被阻塞。

```
用户请求 → Tomcat线程 → 发消息 → 连不上MQ → 重试等1秒 → 再试 → 再等1秒 → 失败
                                                    ↑
                                            3-4秒线程一直被占着
```

**建议**：
- 性能要求高 → 关掉重试，靠后面的 Confirm + 业务补偿兜底
- 要用重试 → 合理配置 `max-attempts` 和 `initial-interval`，缩短阻塞时间
- 最佳实践 → 发消息扔到异步线程里，主流程不阻塞

```java
CompletableFuture.runAsync(() -> {
    rabbitTemplate.convertAndSend("exchange", "key", message);
});
```

### 2.2 Publisher Confirm（发布确认）

解决"消息没到 Exchange"的问题。

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated # 开启publisher confirm机制
```

三种模式：
- `none`：关闭 confirm 机制
- `simple`：同步阻塞等待 MQ 的回执（性能差）
- `correlated`：MQ 异步回调返回回执（推荐）

#### Confirm 回调

每个消息发送时定义，通过 `CorrelationData` 获取回执：

```java
@Test
void testPublisherConfirm() {
    CorrelationData cd = new CorrelationData();
    cd.getFuture().addCallback(new ListenableFutureCallback<CorrelationData.Confirm>() {
        @Override
        public void onFailure(Throwable ex) {
            log.error("send message fail", ex);
        }
        @Override
        public void onSuccess(CrelationData.Confirm result) {
            if(result.isAck()){
                log.info("发送消息成功，收到 ack!");
            }else{
                log.error("发送消息失败，收到 nack, reason : {}", result.getReason());
            }
        }
    });
    rabbitTemplate.convertAndSend("hmall.direct", "q", "hello", cd);
}
```

#### 实际生产建议

开启 Confirm 比较消耗 MQ 性能，一般不建议开启。只有对消息可靠性要求非常高的业务才需要，而且只需要处理 `ack=false` 的情况：

```java
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
    if (!ack) {
        log.error("消息发送到交换机失败，原因：{}", cause);
    }
});
```

### 2.3 Publisher Return（退回回调）

解决"消息到了 Exchange 但路由不到 Queue"的问题。

```yaml
spring:
  rabbitmq:
    publisher-returns: true # 开启publisher return机制
    template:
      mandatory: true # 消息无法路由到队列时退回给生产者
```

#### Return 回调

每个 `RabbitTemplate` 只能配置一个，在配置类中统一设置：

```java
@Slf4j
@AllArgsConstructor
@Configuration
public class MqConfig {
    private final RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init(){
        rabbitTemplate.setReturnsCallback(returned -> {
            log.error("触发return callback,");
            log.debug("exchange: {}", returned.getExchange());
            log.debug("routingKey: {}", returned.getRoutingKey());
            log.debug("message: {}", returned.getMessage());
            log.debug("replyCode: {}", returned.getReplyCode());
            log.debug("replyText: {}", returned.getReplyText());
        });
    }
}
```

### 2.4 三者的关系

```
生产者发送消息
    │
    ├─ 连不上MQ？ ──→ 发送者重连（retry.enabled）
    │
    ▼
消息到达Exchange
    │
    ├─ Exchange不存在？ ──→ Confirm回调 ack=false
    │
    ▼
消息从Exchange路由到Queue
    │
    ├─ 路由不到队列？ ──→ Return回调（mandatory=true）
    │
    ▼
消息进入Queue，等待消费
```

### 2.5 测试验证

发送消息到 `hmall.direct` 交换机，路由键 `q`（不存在的路由键）：

```
ERROR ... 触发return callback,        ← 路由键 q 找不到队列，消息被退回
INFO  ... 发送消息成功，收到 ack!       ← 消息到达了 hmall.direct 交换机
```

---

## 三、MQ 可靠性

消息到达 MQ 后，如果 MQ 不能及时保存，也会导致消息丢失。

### 3.1 数据持久化

| 组件 | 不持久化的后果 | 持久化方式 |
|---|---|---|
| 交换机 | MQ 重启后交换机消失 | `durable=true` |
| 队列 | MQ 重启后队列消失 | `durable=true` |
| 消息 | MQ 重启后消息没了 | `deliveryMode=2` |

**Spring AMQP 默认行为已经是持久化的**，用 `@QueueBinding` 注解声明的队列和交换机默认 `durable=true`，发送的消息默认 `deliveryMode=PERSISTENT`。

#### 持久化 + Confirm 的协作

MQ 收到消息后不是立刻写磁盘，而是**攒一批（约 100ms）再写**。所以 ACK 有延迟，建议用 `correlated`（异步回调）而不是 `simple`（同步阻塞等待）。

### 3.2 LazyQueue（惰性队列）

解决消息堆积时内存不够用的问题。

**问题**：消息堆积 → 内存爆了 → MQ 触发 PageOut → 阻塞队列 → 新消息进不来 → 整个系统卡死

**LazyQueue 的做法**：消息直接写磁盘，消费者要读时才加载到内存

| 对比 | 普通队列 | LazyQueue |
|---|---|---|
| 消息存储 | 先内存，满了再刷盘 | 直接磁盘 |
| 消费时 | 内存里直接读 | 从磁盘加载到内存 |
| 消息堆积时 | 内存爆 → PageOut → 阻塞 | 没影响，本来就在磁盘 |

#### 配置方式

**代码声明**：
```java
@Bean
public Queue lazyQueue(){
    return QueueBuilder
            .durable("lazy.queue")
            .lazy() // 开启Lazy模式
            .build();
}
```

**注解方式**：
```java
@RabbitListener(queuesToDeclare = @Queue(
        name = "lazy.queue",
        durable = "true",
        arguments = @Argument(name = "x-queue-mode", value = "lazy")
))
public void listenLazyQueue(String msg){
    log.info("接收到 lazy.queue的消息：{}", msg);
}
```

**已有队列改模式（Policy）**：
```bash
rabbitmqctl set_policy Lazy "^lazy-queue$" '{"queue-mode":"lazy"}' --apply-to queues
```

#### 版本说明

- 3.6.0 ~ 3.11：可选，需要手动设置
- **3.12+**：默认就是 LazyQueue

---

## 四、消费者可靠性

RabbitMQ 向消费者投递消息后，需要知道消费者的处理状态。

### 4.1 Consumer ACK（消费者确认）

三种回执：

| 回执 | 含义 | RabbitMQ 的动作 |
|---|---|---|
| ack | 处理成功 | 从队列删除消息 |
| nack | 处理失败 | 重新投递消息 |
| reject | 处理失败且拒绝 | 从队列删除消息（不重试） |

三种确认模式：

| 模式 | 配置值 | 行为 | 适用场景 |
|---|---|---|---|
| none | `acknowledge-mode: none` | 消息投递就立刻 ack | ❌ 不安全 |
| manual | `acknowledge-mode: manual` | 手动调 basicAck/basicNack | 灵活但有业务入侵 |
| auto | `acknowledge-mode: auto` | Spring 自动判断 | ✅ 推荐 |

#### auto 模式的判断逻辑

```
消费者处理消息
    │
    ├─ 正常执行 → 自动返回 ack
    │
    ├─ 业务异常（RuntimeException）→ 自动返回 nack（重新投递）
    │
    └─ 消息本身有问题（MessageConversionException等）→ 自动返回 reject（不重试）
```

**reject 的常见异常**（消息本身有问题，重试也没用）：
- `MessageConversionException` — 消息格式转换失败
- `MethodArgumentNotValidException` — 参数校验失败
- `NoSuchMethodException` — 找不到处理方法
- `ClassCastException` — 类型转换错误

#### 踩坑：无限循环

auto 模式下，普通业务异常（RuntimeException）会 nack → 消息重新入队 → 又被消费 → 又抛异常 → 又 nack → **无限循环**！

### 4.2 失败重试机制

解决 nack 无限循环的问题。在消费者本地重试 N 次，失败了再处理。

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: auto
        retry:
          enabled: true # 开启消费者失败重试
          initial-interval: 1000ms # 初识的失败等待时长为1秒
          multiplier: 1 # 失败的等待时长倍数
          max-attempts: 3 # 最大重试次数
          stateless: true # true无状态；false有状态。如果业务中包含事务，这里改为false
```

**效果**：
```
第1次消费 → 异常 → 等1秒重试
第2次重试 → 异常 → 等1秒重试
第3次重试 → 异常 → 放弃，消息被 reject
```

消息不会重新入队，3 次本地重试失败后直接丢弃。

### 4.3 失败处理策略

重试耗尽后的消息处理策略，由 `MessageRecoverer` 接口定义：

| 实现类 | 行为 |
|---|---|
| `RejectAndDontRequeueRecoverer` | 直接 reject，丢弃消息（默认） |
| `ImmediateRequeueMessageRecoverer` | 返回 nack，消息重新入队 |
| `RepublishMessageRecoverer` | 将失败消息投递到指定的交换机（推荐） |

#### RepublishMessageRecoverer 实现

失败消息投递到死信队列，后续人工处理：

```java
@Configuration
@ConditionalOnProperty(name = "spring.rabbitmq.listener.simple.retry.enabled", havingValue = "true")
public class ErrorMessageConfig {

    @Bean
    public DirectExchange errorMessageExchange(){
        return new DirectExchange("error.direct");
    }

    @Bean
    public Queue errorQueue(){
        return new Queue("error.queue", true);
    }

    @Bean
    public Binding errorBinding(Queue errorQueue, DirectExchange errorMessageExchange){
        return BindingBuilder.bind(errorQueue).to(errorMessageExchange).with("error");
    }

    @Bean
    public MessageRecoverer republishMessageRecoverer(RabbitTemplate rabbitTemplate){
        return new RepublishMessageRecoverer(rabbitTemplate, "error.direct", "error");
    }
}
```

**测试效果**：
```
spring 消费者接收到消息：【hello, spring amqp!】  ← 第1次
spring 消费者接收到消息：【hello, spring amqp!】  ← 第2次重试
spring 消费者接收到消息：【hello, spring amqp!】  ← 第3次重试
Republishing failed message to exchange 'error.direct'  ← 重试耗尽，投递到 error.queue
```

---

## 五、业务幂等性

同一个业务，执行一次或多次对业务状态的影响是一致的。

### 问题场景

```
1. 用户支付成功 → 订单状态 → 已支付 ✅
2. 网络故障，生产者没收到ACK → 重新投递消息
3. 用户退款 → 订单状态 → 已退款 ✅
4. 重投的消息被消费 → 订单状态 → 已支付 ❌（退款白退了！）
```

### 5.1 唯一消息ID

每条消息都生成唯一 ID，消费者处理前检查是否已处理过。

```java
@Bean
public MessageConverter messageConverter(){
    Jackson2JsonMessageConverter jjmc = new Jackson2JsonMessageConverter();
    jjmc.setCreateMessageIds(true); // 自动创建消息ID
    return jjmc;
}
```

消费者中使用：
```java
@RabbitListener(queues = "simple.queue")
public void listenSimpleQueue(Message msg) {
    String messageId = msg.getMessageProperties().getMessageId();
    if (isProcessed(messageId)) {
        return; // 已处理过，直接ACK
    }
    doBusiness(msg);
    markProcessed(messageId);
}
```

### 5.2 业务状态判断（推荐）

基于业务本身的逻辑或状态来判断是否重复，不需要额外存储。

**先查再改（有线程安全问题）**：
```java
Order old = getById(orderId);
if (old == null || old.getStatus() != 1) {
    return; // 订单不存在或已处理
}
// 执行更新
```

**WHERE 条件过滤（推荐，原子操作）**：
```java
lambdaUpdate()
        .set(Order::getStatus, 2)
        .set(Order::getPayTime, LocalDateTime.now())
        .eq(Order::getId, orderId)
        .eq(Order::getStatus, 1) // 只有状态=1才更新
        .update();
```

等价 SQL：
```sql
UPDATE `order` SET status = 2, pay_time = ? WHERE id = ? AND status = 1
```

如果 `status != 1`，SQL 匹配不到数据，更新 0 行，天然幂等。

### 两种方案对比

| 方案 | 需要改数据库吗 | 通用性 | 实现复杂度 |
|---|---|---|---|
| 唯一消息ID | ✅ 需要（存消息ID） | 通用 | 中 |
| 业务状态判断 | ❌ 不需要 | 特定业务 | **低** |

---

## 六、延迟消息

在一段时间以后才执行的任务，利用 MQ 的延迟消息实现。

### 典型场景

订单支付超时 30 分钟未支付 → 取消订单 → 释放库存

### 6.1 TTL + 死信队列

```
Publisher → ttl.fanout → ttl.queue（无消费者，设TTL=5秒）
                              ↓ 5秒后消息过期
                           死信交换机 hmall.direct → direct.queue1 → Consumer
```

**注意**：RabbitMQ 的消息过期是基于追溯方式实现的，消息在**队首时**才会被处理。队列堆积多时，过期时间不准确。

### 6.2 DelayExchange 插件（推荐）

安装 `rabbitmq_delayed_message_exchange` 插件后，直接实现延迟消息。

#### 声明延迟交换机

```java
@Bean
public DirectExchange delayExchange(){
    return ExchangeBuilder
            .directExchange("delay.direct")
            .delayed() // 设置delay的属性为true
            .durable(true)
            .build();
}

@Bean
public Queue delayedQueue(){
    return new Queue("delay.queue");
}

@Bean
public Binding delayQueueBinding(){
    return BindingBuilder.bind(delayedQueue()).to(delayExchange()).with("delay");
}
```

#### 发送延迟消息

```java
@Test
void testPublisherDelayMessage() {
    String message = "hello, delayed message";
    rabbitTemplate.convertAndSend("delay.direct", "delay", message, new MessagePostProcessor() {
        @Override
        public Message postProcessMessage(Message message) throws AmqpException {
            message.getMessageProperties().setDelay(5000); // 延迟5秒
            return message;
        }
    });
}
```

#### 测试效果

```
消息发送：约 21:38:49
消息接收：21:38:54
延迟：约 5 秒 ✅
```

#### 注意事项

延迟插件内部维护本地数据库表，使用 Erlang Timers 实现计时：
- 延迟时间过长 → 堆积的延迟消息多 → CPU 开销大
- 延迟时间会有误差
- **不建议设置延迟时间过长的延迟消息**

| 场景 | 推荐方案 |
|---|---|
| 短延迟（几秒~几分钟） | DelayExchange 插件 ✅ |
| 长延迟（几小时~几天） | 定时任务轮询数据库 |

---

## 七、兜底方案

虽然用了各种机制保证消息可靠性，但也不能保证 100%。兜底方案：**定时任务定期查询支付状态**。

```
交易服务 → 每隔20秒查询支付状态 → 发现已支付 → 更新订单状态
```

### 完整的消息可靠性保证体系

```
第一层：生产者端
├─ 发送者重试（网络抖动）
├─ Publisher Confirm（消息到交换机）
└─ Publisher Return（消息到队列）

第二层：Broker端
├─ 交换机持久化
├─ 队列持久化
├─ 消息持久化
└─ LazyQueue（消息堆积）

第三层：消费者端
├─ 手动/自动ACK
├─ 失败重试（本地重试N次）
├─ 死信队列（失败消息不丢）
└─ 业务幂等性（防重复消费）

兜底方案：定时任务
└─ 定期主动查询支付状态，确保最终一致
```

**一句话总结**：MQ通知 + 定时任务兜底 = 最终一致性

---

## 八、关联笔记

- [[SpringAMQP入门与整合]]
- [[RabbitMQ基础篇]]

## 九、代码变更

### publisher 模块

- 修改：`publisher/src/main/resources/application.yml` — 添加重试、Confirm、Return 配置
- 新增：`publisher/src/main/java/com/itheima/publisher/config/MqConfig.java` — ReturnCallback 配置
- 修改：`publisher/src/test/java/com/itheima/publisher/amqp/SpringAmqpTest.java` — 添加 Confirm 和延迟消息测试

### consumer 模块

- 修改：`consumer/src/main/resources/application.yml` — 添加 ACK 模式、失败重试配置
- 新增：`consumer/src/main/java/com/itheima/consumer/config/DlxConfig.java` — 错误消息交换机/队列 + RepublishMessageRecoverer
- 新增：`consumer/src/main/java/com/itheima/consumer/config/DelayExchangeConfig.java` — 延迟交换机配置
- 修改：`consumer/src/main/java/com/itheima/consumer/listener/SpringRabbitListener.java` — 添加延迟消息监听器
