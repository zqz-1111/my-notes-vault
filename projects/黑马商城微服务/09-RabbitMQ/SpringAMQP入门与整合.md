---
date: 2026-07-10
tags: [RabbitMQ, SpringAMQP, SpringBoot, 黑马商城]
---

# SpringAMQP 入门与整合

## 一、写在前面

[[RabbitMQ架构与Docker部署|RabbitMQ]] 已经部署好了，但实际开发不会在控制台手动收发消息，而是通过代码操作。RabbitMQ 采用 AMQP 协议，任何语言只要遵循 AMQP 就能与它交互。Java 生态下，Spring 官方提供了一套封装好的工具 —— **SpringAMQP**。

**学完这篇你能掌握：**
- SpringAMQP 是什么、解决了什么问题
- SpringAMQP 的三大核心功能
- SpringBoot 整合 RabbitMQ 的配置步骤

## 二、为什么不用 RabbitMQ 官方 Java 客户端

RabbitMQ 官方提供了 Java 客户端，但编码复杂：
- 手动管理连接和 Channel
- 手动声明队列、交换机、绑定关系
- 手动处理异常、重连、确认机制

代码量大且容易出错。**SpringAMQP 帮你封装好了这些，你只关心"发什么"和"收什么"。**

## 三、SpringAMQP 三大核心功能

| 功能 | 代码层面 | 替代了什么 |
|---|---|---|
| 自动声明队列、交换机、绑定 | `@Bean` 声明 或 配置文件 | 手动去控制台点点点 |
| 注解监听器异步收消息 | `@RabbitListener(queues="xxx")` | 手动写 `channel.basicConsume()` |
| RabbitTemplate 发消息 | `rabbitTemplate.convertAndSend()` | 手动写 `channel.basicPublish()` |

### 3.1 自动声明队列、交换机及绑定关系

SpringAMQP 启动时会自动检查代码中声明的 Queue、Exchange、Binding，如果 RabbitMQ 里没有就自动创建，不用手动去控制台操作。

### 3.2 基于注解的监听器模式

用 `@RabbitListener` 注解标记方法，SpringAMQP 会自动监听指定队列，有消息就调用方法处理，完全异步：

```java
@RabbitListener(queues = "simple.queue")
public void listenSimpleQueueMessage(String msg) {
    log.info("收到消息：{}", msg);
}
```

### 3.3 RabbitTemplate 消息发送模板

封装了底层连接管理，一行代码发消息：

```java
rabbitTemplate.convertAndSend("simple.queue", "Hello RabbitMQ!");
```

## 四、SpringBoot 整合 RabbitMQ

### 4.1 引入依赖

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 4.2 配置连接信息

```yaml
# application.yaml
spring:
  rabbitmq:
    host: 192.168.188.130   # 虚拟机IP（RabbitMQ所在服务器）
    port: 5672               # AMQP端口（不是15672管理端口）
    username: itheima
    password: 123321
    virtual-host: /          # 默认vhost
```

> **注意：** `port` 是 **5672**（AMQP 协议端口），不是 15672（管理控制台端口）。这两个端口用途不同：
> - **5672**：应用收发消息用
> - **15672**：浏览器访问管理后台用

### 4.3 自动装配

引入依赖 + 写好配置后，SpringBoot 自动装配生效，直接注入 `RabbitTemplate` 就能用：

```java
@Autowired
private RabbitTemplate rabbitTemplate;
```

## 五、踩坑避坑指南

### 逻辑坑

**坑 1：端口搞混**
- 问题：配置写成 `port: 15672`，连接超时
- 原因：15672 是 Web 管理端口，AMQP 协议走 5672
- 解决：业务端口用 5672

**坑 2：virtual-host 不匹配**
- 问题：连接成功但发消息报错 `NOT_FOUND - vhost xxx not found`
- 原因：代码里配置的 vhost 在 RabbitMQ 里不存在
- 解决：默认用 `/`，自定义 vhost 需要先在控制台创建

### 报错坑

**坑 3：Connection refused**
- 报错：`java.net.ConnectException: Connection refused`
- 原因：RabbitMQ 容器没启动 / 端口没映射 / IP 地址写错
- 解决：`docker ps | grep mq` 确认容器在运行，检查端口映射和 IP

## 六、全文总结

| 核心知识点 | 要点 |
|---|---|
| SpringAMQP 是什么 | Spring 官方基于 AMQP 协议封装的消息收发工具 |
| 三大功能 | 自动声明队列交换机、注解监听器收消息、RabbitTemplate 发消息 |
| 整合步骤 | 加 starter-amqp 依赖 → yaml 配置连接信息 → 注入 RabbitTemplate |
| 端口区分 | 5672 业务收发消息，15672 管理控制台 |

## 七、下一步

SpringAMQP 整合完成，接下来进入实操：
- **基本消息队列**：最简单的点对点收发（一个生产者 → 一个队列 → 一个消费者）
- **Work Queue**：多个消费者竞争消费同一个队列
- **交换机类型**：Fanout / Direct / Topic 三种路由策略

## 八、关联笔记

- [[09-RabbitMQ/README|RabbitMQ 目录]]
- [[RabbitMQ架构与Docker部署|RabbitMQ 架构与 Docker 部署]]
- [[初识MQ：同步调用问题与异步架构|同步调用问题与异步架构]]

## 九、代码变更

- 新增：`projects/黑马商城微服务/09-RabbitMQ/SpringAMQP入门与整合.md` — SpringAMQP 三大功能、SpringBoot 整合配置
