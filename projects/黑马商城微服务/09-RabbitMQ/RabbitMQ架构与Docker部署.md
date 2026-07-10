---
date: 2026-07-10
tags: [RabbitMQ, Docker, 消息队列, 黑马商城]
---

# RabbitMQ 架构与 Docker 部署

## 一、写在前面

上一篇我们搞清楚了为什么要用 MQ（[[初识MQ：同步调用问题与异步架构|同步调用三大问题]]），这一篇正式进入 RabbitMQ 的世界。

**学完这篇你能掌握：**
- RabbitMQ 的五大核心概念（Publisher、Consumer、Queue、Exchange、Virtual Host）
- 消息在 RabbitMQ 内部的完整流转链路
- Docker 一键部署 RabbitMQ 并验证

## 二、RabbitMQ 架构概念

### 2.1 消息流转全链路

```
Publisher → Exchange → Queue → Consumer
  生产者      交换机     队列      消费者
```

生产者把消息发给交换机，交换机根据路由规则把消息塞到对应的队列，消费者从队列里取消息。

### 2.2 五大核心概念

| 概念 | 角色 | 一句话解释 |
|---|---|---|
| **Publisher** | 生产者 | 发送消息的一方，业务代码调用 `rabbitTemplate.convertAndSend()` |
| **Exchange** | 交换机 | 负责消息路由，决定消息投递到哪个 Queue |
| **Queue** | 队列 | 消息暂存的地方，消费者从这里取消息 |
| **Consumer** | 消费者 | 接收消息的一方，用 `@RabbitListener` 监听队列 |
| **Virtual Host** | 虚拟主机 | 数据隔离空间，类似 MySQL 的 database |

### 2.3 Virtual Host 的隔离作用

```
Virtual Host: /trade
  ├── exchange: trade.direct
  └── queue: trade.order.queue

Virtual Host: /pay
  ├── exchange: pay.direct
  └── queue: pay.result.queue
```

两个 vhost 里的 exchange 和 queue 即使名字相同也完全独立，互不影响。实际项目中可以按业务模块划分 vhost。

### 2.4 与已学概念的类比

| RabbitMQ 概念 | 对应概念 | 共同作用 |
|---|---|---|
| Virtual Host | Nacos 的 namespace | 数据隔离 |
| Exchange + Queue | Sentinel 的规则 | 路由 + 限流 |
| Publisher / Consumer | OpenFeign 调用方/被调用方 | 解耦通信 |

## 三、Docker 部署 RabbitMQ

### 3.1 部署命令

```bash
docker run \
 -e RABBITMQ_DEFAULT_USER=itheima \
 -e RABBITMQ_DEFAULT_PASS=123321 \
 -v mq-plugins:/plugins \
 --name mq \
 --hostname mq \
 -p 15672:15672 \
 -p 5672:5672 \
 --network hm-net \
 --restart always \
 -d \
 rabbitmq:3.8-management
```

### 3.2 命令解读

| 参数 | 作用 |
|---|---|
| `-e RABBITMQ_DEFAULT_USER=itheima` | 设置管理员用户名 |
| `-e RABBITMQ_DEFAULT_PASS=123321` | 设置管理员密码 |
| `-v mq-plugins:/plugins` | 数据卷挂载，持久化插件目录 |
| `--name mq` | 容器名称 |
| `--hostname mq` | 容器主机名，RabbitMQ 内部用来标识节点 |
| `-p 15672:15672` | 管理控制台端口（Web UI） |
| `-p 5672:5672` | AMQP 协议端口（应用收发消息用） |
| `--network hm-net` | 加入 hm-net 容器网络，与其他微服务互通 |
| `--restart always` | 容器挂了自动重启，服务器重启后也自动启动 |
| `rabbitmq:3.8-management` | 镜像版本，带 management 插件（有 Web 控制台） |

### 3.3 前置条件

确保 `hm-net` 网络存在：

```bash
# 检查网络
docker network ls | grep hm-net

# 没有就创建
docker network create hm-net
```

### 3.4 验证部署

```bash
# 查看容器是否正常运行
docker ps | grep mq

# 访问管理控制台
# 浏览器打开 http://虚拟机IP:15672
# 用户名: itheima  密码: 123321
```

## 四、踩坑避坑指南

### 逻辑坑

**坑 1：hm-net 网络不存在**
- 问题：直接运行 docker run 报错 `network hm-net not found`
- 原因：之前没创建过这个网络
- 解决：先 `docker network create hm-net`

**坑 2：端口没映射完整**
- 问题：应用连不上 RabbitMQ
- 原因：只映射了 15672（控制台），没映射 5672（AMQP）
- 解决：两个端口都要映射，5672 是业务收发消息用的

### 报错坑

**坑 3：容器名冲突**
- 报错：`docker: Error response from daemon: Conflict. The container name "/mq" is already in use`
- 解决：`docker rm -f mq` 删掉旧容器，或者换个名字

## 五、全文总结

| 核心知识点 | 要点 |
|---|---|
| 五大概念 | Publisher → Exchange → Queue → Consumer，Virtual Host 做隔离 |
| 消息流转 | 生产者发到交换机 → 交换机路由到队列 → 消费者从队列取消息 |
| Docker 部署 | 两个端口 5672+15672、环境变量设账号密码、加入 hm-net 网络 |
| --restart always | 生产环境必加，容器挂了自动拉起 |

## 六、关联笔记

- [[09-RabbitMQ/README|RabbitMQ 目录]]
- [[初识MQ：同步调用问题与异步架构|同步调用问题与异步架构]]
- [[02-Docker/README|Docker 目录]]

## 七、代码变更

- 新增：`projects/黑马商城微服务/09-RabbitMQ/RabbitMQ架构与Docker部署.md` — 五大核心概念、消息流转链路、Docker 部署命令
