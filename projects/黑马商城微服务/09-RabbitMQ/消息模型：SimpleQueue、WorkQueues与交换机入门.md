---
date: 2026-07-10
tags: [RabbitMQ, 消息队列, 黑马商城, SpringAMQP]
---

# 消息模型：SimpleQueue、WorkQueues 与交换机入门

## 一、写在前面

1. 为什么学这个：掌握了 RabbitMQ 架构和 SpringAMQP 整合之后，下一步就是搞清楚消息怎么发、怎么收、怎么分发。消息模型决定了消息从生产者到消费者的流转方式。
2. 学完这篇你能掌握：SimpleQueue 基本收发、WorkQueues 多消费者分摊、能者多劳原理、以及交换机的三种核心类型。

## 二、SimpleQueue 基本消息模型

最简单的点对点模型：一个生产者 → 一个队列 → 一个消费者。

### 2.1 生产者发送消息

利用 `RabbitTemplate.convertAndSend()` 直接发送到队列：

```java
@SpringBootTest
public class SpringAmqpTest {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Test
    public void testSimpleQueue() {
        // 队列名称
        String queueName = "simple.queue";
        // 消息
        String message = "hello, spring amqp!";
        // 发送消息
        rabbitTemplate.convertAndSend(queueName, message);
    }
}
```

### 2.2 消费者接收消息

用 `@RabbitListener` 注解声明监听哪个队列，消息到达时自动调用方法：

```java
@Component
public class SpringRabbitListener {

    @RabbitListener(queues = "simple.queue")
    public void listenSimpleQueueMessage(String msg) throws InterruptedException {
        System.out.println("spring 消费者接收到消息：【" + msg + "】");
    }
}
```

> **注意：** 队列需要提前在 RabbitMQ 控制台创建，否则消息发不出去。

## 三、WorkQueues 任务模型

### 3.1 解决什么问题

当消息处理比较耗时，生产速度远大于消费速度时，消息会堆积。WorkQueues 让多个消费者绑定同一个队列，共同消费，提高处理速度。

### 3.2 发送端：模拟消息堆积

循环发送 50 条消息，每 20ms 一条（每秒 50 条）：

```java
@Test
public void testWorkQueue() throws InterruptedException {
    String queueName = "work.queue";
    String message = "hello, message_";
    for (int i = 0; i < 50; i++) {
        rabbitTemplate.convertAndSend(queueName, message + i);
        Thread.sleep(20);
    }
}
```

### 3.3 接收端：两个消费者绑定同一队列

```java
@RabbitListener(queues = "work.queue")
public void listenWorkQueue1(String msg) throws InterruptedException {
    System.out.println("消费者1接收到消息：【" + msg + "】" + LocalTime.now());
    Thread.sleep(20); // 处理快
}

@RabbitListener(queues = "work.queue")
public void listenWorkQueue2(String msg) throws InterruptedException {
    System.err.println("消费者2........接收到消息：【" + msg + "】" + LocalTime.now());
    Thread.sleep(200); // 处理慢
}
```

### 3.4 默认行为：轮询分配

不加任何配置时，RabbitMQ 默认**轮询（round-robin）**分配消息——不管消费者处理速度快慢，一人一半。

### 3.5 能者多劳（prefetch）

默认轮询的问题：处理慢的消费者会堆积消息，因为消息已经预取（prefetch）到它手里了。

解决方案：在 `application.yml` 中配置 `prefetch: 1`：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 1 # 每次只能获取一条消息，处理完成才能获取下一个
```

**原理：** 设置 prefetch = 1 后，消费者处理完当前消息才能拿下一条。处理快的消费者会更快地"空出手"来抢下一条，自然就多干活了。

**实测效果：**
- 消费者1（20ms）：收到 43 条
- 消费者2（200ms）：收到 7 条

### 3.6 WorkQueues 小结

| 特性 | 说明 |
|---|---|
| 绑定方式 | 多个消费者绑定同一个队列 |
| 消息分配 | 同一条消息只会被一个消费者处理 |
| 默认策略 | 轮询分配，一人一半 |
| 能者多劳 | 设置 `prefetch: 1`，处理快的消费者多干活 |

## 四、交换机（Exchange）入门

### 4.1 为什么需要交换机

之前的消息模型都是生产者直接发到队列。引入交换机后，消息流转变成：

```
Publisher → Exchange → Queue → Consumer
```

交换机的角色：
- 接收生产者发送的消息
- 根据类型和规则，决定消息路由到哪些队列

> **重要：** 交换机只负责转发，不具备存储能力。如果没有队列与交换机绑定，或者没有符合路由规则的队列，消息会丢失！

### 4.2 三种交换机类型

| 类型 | 行为 | 适用场景 |
|---|---|---|
| **Fanout（广播）** | 把消息交给所有绑定到交换机的队列 | 通知、广播 |
| **Direct（路由）** | 基于 RoutingKey 精确匹配，发给指定队列 | 精确路由 |
| **Topic（通配符）** | 类似 Direct，但 RoutingKey 支持通配符（`*` 匹配一个词，`#` 匹配零或多个词） | 灵活路由 |

> 还有一种 Headers 类型，基于消息头匹配，用得较少，课程不讲。

### 4.3 消息发送模式的变化

引入交换机后，生产者不再直接发消息到队列，而是发给交换机。队列必须与交换机绑定才能收到消息。

## 五、踩坑避坑指南

### 踩坑1：队列名不一致
- 问题：`testWorkQueue` 里写的是 `simple.queue`，但消费者监听的是 `work.queue`
- 后果：消息发到了 simple.queue，work.queue 的消费者收不到任何消息
- 正确做法：发送端和接收端的队列名必须一致

### 踩坑2：队列未创建
- 问题：代码里引用的队列在 RabbitMQ 控制台没有创建
- 后果：消息发送失败或消费者监听无效
- 正确做法：先在 RabbitMQ 控制台（http://IP:15672）创建对应的队列

## 六、代码变更

- 新增：`publisher/src/test/java/com/itheima/publisher/amqp/SpringAmqpTest.java` — 发送消息测试类
- 新增：`consumer/src/main/java/com/itheima/consumer/listener/SpringRabbitListener.java` — 消息监听器
- 修改：`publisher/src/main/resources/application.yml` — 添加 RabbitMQ 连接配置
- 修改：`consumer/src/main/resources/application.yml` — 添加 RabbitMQ 连接配置 + prefetch: 1

## 七、关联笔记

- [[SpringAMQP入门与整合]] — SpringAMQP 基础整合配置
- [[RabbitMQ架构与Docker部署]] — 五大核心概念、Docker 部署
- [[初识MQ：同步调用问题与异步架构]] — 同步异步概念、MQ 选型
