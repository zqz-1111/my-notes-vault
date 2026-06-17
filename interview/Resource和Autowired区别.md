---
date: 2026-06-17
tags: [interview, Spring]
---

# @Resource 和 @Autowired 有什么区别？

## 问题

Spring 中注入 Bean 时，`@Resource` 和 `@Autowired` 都能用，它们有什么区别？什么时候用哪个？

## 答案

| 对比项 | @Resource | @Autowired |
|---|---|---|
| 来源 | JDK（`javax.annotation`） | Spring（`org.springframework`） |
| 注入方式 | **先按名称**匹配，再按类型 | **先按类型**匹配 |
| 指定具体 bean | `@Resource(name="xxx")` | `@Autowired` + `@Qualifier("xxx")` |
| required 控制 | 没有，找不到就报错 | `@Autowired(required=false)` 可选注入 |

**核心区别：** `@Resource` 先按名字找，`@Autowired` 先按类型找。

### 同类型只有一个 Bean 时

两个效果完全一样，随便用。

### 同类型有多个 Bean 时

```java
@Bean public DataSource masterDataSource() { ... }
@Bean public DataSource slaveDataSource() { ... }
```

```java
@Resource
private DataSource masterDataSource;  // ✅ 按名字找到

@Autowired
@Qualifier("masterDataSource")
private DataSource masterDataSource;  // ✅ 需要额外加 @Qualifier
```

## 深入追问

- **追问 1：Spring Boot 项目推荐用哪个？** 推荐 `@Autowired`，和 Spring 生态一致。Java 11+ 中 `@Resource` 所在的 `javax.annotation` 包被移除了，需要额外引依赖。
- **追问 2：`@Inject` 和它们什么关系？** `@Inject` 是 JSR-330 标准，功能和 `@Autowired` 类似，按类型注入，需要额外引入 `javax.inject` 依赖。

## 来源

hm-dianping 项目中 `ShopTypeServiceImpl` 使用 `@Autowired` 注入 `StringRedisTemplate`，和 `ShopServiceImpl` 中的 `@Resource` 写法不同，引发讨论。
