---
date: 2026-06-20
tags: [RabbitMQ, 消息队列, 黑马商城]
---

# 09-RabbitMQ

## 说明

RabbitMQ 是一个开源的消息中间件，用于实现异步通信、服务解耦、流量削峰等。

## 学习内容

- 消息队列概述 ✅
- RabbitMQ 架构概念 ✅
- RabbitMQ Docker 部署 ✅
- SpringAMQP 整合 ✅
- 基本消息模型 ✅
- Work Queue ✅
- 交换机类型 ✅
- 业务改造：同步改异步 ✅
- 登录信息传递优化 ✅
- 死信队列 ✅
- 延迟队列 ✅
- 订单超时自动取消 ✅
- MQ 工具抽取 ✅

## 笔记列表

- [[初识MQ：同步调用问题与异步架构]] — 同步调用三大问题、异步模型、MQ技术选型对比
- [[RabbitMQ架构与Docker部署]] — 五大核心概念、消息流转链路、Docker部署命令
- [[SpringAMQP入门与整合]] — SpringAMQP三大功能、SpringBoot整合配置
- [[消息模型：SimpleQueue、WorkQueues与交换机入门]] — SimpleQueue收发、WorkQueues多消费者分摊、能者多劳prefetch、交换机三种类型
- [[交换机实战、声明方式与消息转换器]] — Fanout/Direct/Topic代码实战、@Bean与注解两种声明方式、JSON消息转换器配置
- [[业务改造：同步调用改MQ异步通知]] — 支付异步通知、下单异步清理购物车、MQ共享配置抽取Nacos
- [[登录信息传递优化：MQ消息自动携带用户信息]] — MqConfig自动装配、Header传用户、业务代码无感知
- [[死信队列与延迟消息]] — 死信交换机原理、TTL+死信实现延迟、DelayExchange插件、方案对比选型
- [[延迟消息实战：订单超时自动取消]] — 延迟消息完整业务流程、cancelOrder幂等性设计、支付流水二次校验
- [[MQ工具抽取：RabbitMqHelper与消费失败处理]] — RabbitMqHelper三方法、MqConsumeErrorAutoConfiguration、Nacos共享配置
