---
date: 2026-06-17
tags: [interview, SpringBoot]
---

# Spring Boot 自动装配原理

## 问题

`StringRedisTemplate` 没有手动写 `@Bean` 方法，为什么能直接 `@Autowired` 注入？它从哪来的？

## 答案

是 **Spring Boot 自动装配**（Auto Configuration）帮你创建的。

### 流程

```
pom.xml 引入 spring-boot-starter-data-redis
  ↓
Spring Boot 检测到 classpath 下有 Redis 相关类
  ↓
触发 RedisAutoConfiguration 自动配置类
  ↓
自动创建 StringRedisTemplate、RedisTemplate 等 Bean 注册到 IOC 容器
  ↓
你用 @Autowired / @Resource 直接注入就能用
```

### 关键注解

```java
@SpringBootApplication  // 这个注解包含了：
  └── @EnableAutoConfiguration  // 开启自动装配
        └── 读取 META-INF/spring.factories  // 所有自动配置类的注册表
              └── RedisAutoConfiguration    // Redis 的自动配置类
```

## 深入追问

- **追问 1：自动装配可以覆盖吗？** 可以。你自己写一个 `@Bean` 方法返回 `StringRedisTemplate`，就会覆盖自动装配的。Spring Boot 的设计是"有你用你的，没有我给你配"。
- **追问 2：`spring.factories` 是什么？** 是一个配置文件，列出了所有自动配置类的全限定名。Spring Boot 启动时扫描这个文件，逐个判断是否需要激活。
- **追问 3：怎么查看某个 Bean 是自动装配的还是手动写的？** 启动时加 `--debug` 参数，或在 `application.yml` 里配 `debug: true`，控制台会打印自动装配报告（Positive matches / Negative matches）。

## 来源

hm-dianping 项目中 `StringRedisTemplate` 通过 `@Resource` 或 `@Autowired` 注入使用，但代码中找不到手动定义的 `@Bean`，实为 Spring Boot 自动装配提供。
