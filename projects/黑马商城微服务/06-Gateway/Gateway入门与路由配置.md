---
date: 2026-07-05
tags: [SpringCloud, Gateway, 微服务, 黑马商城]
---

# 【Gateway】网关入门与路由配置

## 一、写在前面

1. 微服务拆分后，前端要记 5 个服务地址（item:8081、cart:8082...），维护麻烦、每个服务都要单独做鉴权——Gateway 就是来解决这个问题的
2. 学完这篇你能掌握：Gateway 是什么、路由规则怎么配、断言类型有哪些

## 二、什么是网关

**一句话：网关就是微服务的"统一门卫"。**

类比小区大门：快递员只管送到大门口，门卫根据包裹地址分发到具体楼栋。门卫还能查身份证（鉴权）、限制每分钟进多少人（限流）、登记来访记录（日志）。

```
前端请求 → Gateway(统一入口) → 路由到具体微服务
```

### 网关核心干三件事

| 功能 | 说明 | 项目里的例子 |
|---|---|---|
| **路由** | 根据请求路径转发到对应服务 | `/items/**` → item-service |
| **过滤** | 请求到达前/后做处理 | 校验 Token、记录日志 |
| **集成 Nacos** | 自动发现服务地址，不用写死 IP | 新加服务不用改网关配置 |

## 三、Gateway vs Nginx

| 维度 | Nginx | Gateway |
|---|---|---|
| 语言 | C | Java |
| 谁管 | 运维 | 开发 |
| 配置方式 | nginx.conf | Spring 配置 / Java 代码 |
| 动态路由 | 不方便，改配置要 reload | 热加载，配合 Nacos 自动发现 |
| 与微服务集成 | 不认识 Nacos | 原生对接 Nacos，lb:// 自动负载均衡 |
| 鉴权 | 需要写 Lua 或调外部服务 | 直接写 Java Filter，能调业务代码 |
| 静态资源 | 擅长（高性能文件服务） | 不干这事 |
| 性能 | 极高（C 写的） | 够用（Netty） |

**实际项目两个都用：**

```
Nginx（外层，运维管）        Gateway（内层，开发管）
├── 域名解析                 ├── 路由到微服务
├── HTTPS 证书               ├── Java 鉴权逻辑
├── 静态资源（前端）           ├── 限流/熔断
├── 反向代理到 Gateway        ├── 灰度发布
└── 第一层负载均衡            └── 动态路由（Nacos）
```

**Nginx 是小区大门，Gateway 是每栋楼的门禁。**

## 四、路由配置

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: item               # ① 路由ID，唯一
          uri: lb://item-service # ② 转发目标，lb=负载均衡
          predicates:            # ③ 匹配条件
            - Path=/items/**     # ④ 路径规则
```

| 属性 | 含义 |
|---|---|
| `id` | 路由的唯一标识 |
| `uri` | 转发目标，`lb://` 从 Nacos 获取服务列表并负载均衡 |
| `predicates` | 路由断言，满足条件才走这条路由 |
| `filters` | 路由过滤器（后续讲） |

### 项目完整路由配置

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: item
          uri: lb://item-service
          predicates:
            - Path=/items/**,/search/**
        - id: cart
          uri: lb://cart-service
          predicates:
            - Path=/carts/**
        - id: user
          uri: lb://user-service
          predicates:
            - Path=/users/**,/addresses/**
        - id: trade
          uri: lb://trade-service
          predicates:
            - Path=/orders/**
        - id: pay
          uri: lb://pay-service
          predicates:
            - Path=/pay-orders/**
```

## 五、断言类型

断言就是 `if` 条件，满足了才走这条路由。可以叠加，全部满足才匹配。

### 最常用

```yaml
# Path — 路径匹配
- Path=/items/**

# Method — 限制请求方式
- Method=GET,POST

# Query — 要求 URL 带指定参数
- Query=name        # 匹配 /items?name=手机
- Query=name, Jack  # 值必须是 Jack
```

### 时间类（秒杀场景）

```yaml
# After — 某时间点之后
- After=2026-07-06T00:00:00+08:00[Asia/Shanghai]

# Before — 某时间点之前
- Before=2026-07-06T23:59:59+08:00[Asia/Shanghai]

# Between — 两个时间点之间
- Between=2026-07-06T10:00:00+08:00[Asia/Shanghai], 2026-07-06T12:00:00+08:00[Asia/Shanghai]
```

### 请求头/参数类

```yaml
# Header — 请求头必须包含
- Header=X-Request-Id, \d+

# Cookie — 必须带指定 Cookie
- Cookie=token, .+

# Host — 限制域名
- Host=**.hmall.com
```

### 来源控制

```yaml
# RemoteAddr — 限制 IP
- RemoteAddr=192.168.1.1/24
```

### 权重路由（灰度发布）

```yaml
# 90% 流量到 v1，10% 到 v2
- Weight=item, 90
- Weight=item, 10
```

### 速记表

| 断言 | 控制什么 | 常用程度 |
|---|---|---|
| Path | 请求路径 | ⭐⭐⭐ 必用 |
| Method | GET/POST/PUT/DELETE | ⭐⭐ 常用 |
| Query | URL 参数 | ⭐ 偶尔 |
| Header | 请求头 | ⭐ 偶尔 |
| After/Before/Between | 时间限制 | ⭐ 秒杀场景 |
| Cookie | Cookie | ⭐ 鉴权场景 |
| Host | 域名 | ⭐ 多域名场景 |
| RemoteAddr | IP 地址 | ⭐ 内网限制 |
| Weight | 流量比例 | ⭐ 灰度发布 |

## 六、踩坑避坑指南

### 报错坑 1：Nacos gRPC 端口不通

- 报错信息：`Server check fail, please check server 192.168.150.101, port 9848 is available`
- 原因分析：Nacos 2.x 用两个端口——8848（HTTP）和 9848（gRPC = 8848+1000）。Docker 部署时只映射了 8848，没映射 9848
- 解决方案：Docker 启动时加上 `-p 9848:9848 -p 9849:9849`

### 报错坑 2：Invalid keystore format

- 报错信息：`Cannot load keys from store: class path resource [hmall.jks]` + `Invalid keystore format`
- 原因分析：缺少 `spring-security-crypto` 和 `spring-security-rsa` 依赖，传递依赖拉到了旧版本 `1.0.10.RELEASE`，跟项目用的 `1.0.9.RELEASE` 不兼容
- 解决方案：在 pom.xml 显式加上两个依赖（与 user-service 保持一致）：

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-crypto</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-rsa</artifactId>
    <version>1.0.9.RELEASE</version>
</dependency>
```

## 七、效果验证

| 请求 | 状态码 | 说明 |
|---|---|---|
| `GET /items/1` | 200 | ✅ 白名单放行 |
| `GET /search/list` | 200 | ✅ 白名单放行 |
| `GET /users/login` | 503 | ✅ 白名单放行（user-service 没启动所以 503） |
| `GET /orders` | 401 | ✅ 非白名单，没 Token，被拦截 |
| `GET /carts` | 401 | ✅ 非白名单，没 Token，被拦截 |

## 八、全文总结

- Gateway = 统一入口 + 路由分发 + 公共处理（鉴权/限流/日志）
- 路由三要素：id（名字）、uri（去哪）、predicates（什么条件去）
- Nginx 面向运维管服务器层，Gateway 面向开发管微服务层，实际项目配合使用
- Nacos 2.x 要同时映射 8848 和 9848 端口

## 九、关联笔记

- [[微服务拆分原则与实操]]
- [[OpenFeign快速入门]]
- [[Gateway鉴权过滤器实现]]

## 十、代码变更

- 新增：`hm-gateway/src/main/resources/application.yaml` — 路由配置
