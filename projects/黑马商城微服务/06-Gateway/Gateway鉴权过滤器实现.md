---
date: 2026-07-05
tags: [SpringCloud, Gateway, JWT, 鉴权, 黑马商城]
---

# 【Gateway】鉴权过滤器实现

## 一、写在前面

1. 之前每个微服务各自校验 JWT，秘钥散落各处不安全、代码重复——Gateway 统一鉴权就是解决这个问题
2. 学完这篇你能掌握：Gateway 怎么在转发前做校验、用户信息怎么跨服务传递、GlobalFilter 的完整实现

## 二、鉴权思路分析

### 之前的问题

每个微服务都做登录校验：
- 每个微服务都需要知道 JWT 秘钥 → **不安全**
- 每个微服务重复编写登录校验代码 → **麻烦**

### 网关统一鉴权

- 秘钥只在 Gateway 和 user-service 保存
- 只在 Gateway 开发登录校验功能

### 三个核心问题

| 问题 | 方案 |
|---|---|
| 网关怎么在转发前校验？ | **GlobalFilter**，在 filter() 里写鉴权逻辑 |
| 网关怎么把用户信息传给微服务？ | **请求头**，`request.mutate().header("user-info", userId)` |
| 微服务之间怎么传递？ | **Feign 拦截器**，`RequestInterceptor` 自动从 ThreadLocal 取出塞到下游请求头 |

## 三、配置类

### AuthProperties — 白名单配置

```java
@Data
@ConfigurationProperties(prefix = "hm.auth")
public class AuthProperties {
    private List<String> includePaths;
    private List<String> excludePaths;  // 不需要登录的路径
}
```

### JwtProperties — JWT 秘钥配置

```java
@Data
@ConfigurationProperties(prefix = "hm.jwt")
public class JwtProperties {
    private Resource location;    // 秘钥文件位置
    private String password;      // 秘钥文件密码
    private String alias;         // 秘钥别名
    private Duration tokenTTL;    // Token 有效期
}
```

### SecurityConfig — 秘钥 Bean 配置

```java
@Configuration
@EnableConfigurationProperties(JwtProperties.class)
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }

    @Bean
    public KeyPair keyPair(JwtProperties properties){
        KeyStoreKeyFactory keyStoreKeyFactory =
                new KeyStoreKeyFactory(
                        properties.getLocation(),
                        properties.getPassword().toCharArray());
        return keyStoreKeyFactory.getKeyPair(
                properties.getAlias(),
                properties.getPassword().toCharArray());
    }
}
```

### application.yaml 配置

```yaml
hm:
  jwt:
    location: classpath:hmall.jks
    alias: hmall
    password: hmall123
    tokenTTL: 30m
  auth:
    excludePaths:
      - /search/**
      - /users/login
      - /items/**
```

## 四、AuthGlobalFilter 核心实现

```java
@Component
@RequiredArgsConstructor
@EnableConfigurationProperties(AuthProperties.class)
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    private final JwtTool jwtTool;
    private final AuthProperties authProperties;
    private final AntPathMatcher antPathMatcher = new AntPathMatcher();

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 1.获取Request
        ServerHttpRequest request = exchange.getRequest();
        // 2.判断是否不需要拦截
        if(isExclude(request.getPath().toString())){
            return chain.filter(exchange);  // 白名单直接放行
        }
        // 3.获取请求头中的token
        String token = null;
        List<String> headers = request.getHeaders().get("authorization");
        if (!CollUtils.isEmpty(headers)) {
            token = headers.get(0);
        }
        // 4.校验并解析token
        Long userId = null;
        try {
            userId = jwtTool.parseToken(token);
        } catch (UnauthorizedException e) {
            ServerHttpResponse response = exchange.getResponse();
            response.setRawStatusCode(401);
            return response.setComplete();  // 校验失败返回401
        }
        // 5.传递用户信息到下游微服务
        ServerHttpRequest newRequest = request.mutate()
                .header("user-info", userId.toString())
                .build();
        // 6.放行
        return chain.filter(exchange.mutate().request(newRequest).build());
    }

    private boolean isExclude(String antPath) {
        for (String pathPattern : authProperties.getExcludePaths()) {
            if(antPathMatcher.match(pathPattern, antPath)){
                return true;
            }
        }
        return false;
    }

    @Override
    public int getOrder() {
        return 0;  // 数字越小越先执行
    }
}
```

## 五、执行流程详解

### 核心对象

| 对象 | 是什么 | 干嘛用 |
|---|---|---|
| `exchange` | 请求+响应的包装体 | 拿请求、写响应 |
| `request` | 请求对象 | 取路径、取头、取参数 |
| `chain` | 过滤器链 | 调 `chain.filter()` 放行到下一个过滤器 |

### 流程图

```
GET /orders (无Token)
  ↓
AuthGlobalFilter.filter()
  ├─ isExclude("/orders") → false（不在白名单）
  ├─ getHeader("authorization") → null
  ├─ jwtTool.parseToken(null) → 抛 UnauthorizedException
  ├─ catch → setStatusCode(401)
  └─ return response.setComplete()  ← 返回 401

GET /orders (带Token)
  ↓
AuthGlobalFilter.filter()
  ├─ isExclude("/orders") → false
  ├─ getHeader("authorization") → "eyJhbGci..."
  ├─ jwtTool.parseToken("eyJhbGci...") → userId = 12345
  ├─ request.mutate().header("user-info", "12345")
  └─ chain.filter()  ← 放行，转发到 trade-service

GET /items/1 (无Token)
  ↓
AuthGlobalFilter.filter()
  ├─ isExclude("/items/1") → true（在白名单）
  └─ chain.filter()  ← 直接放行，不校验 Token
```

### 关键点：request.mutate()

`request.mutate()` 不是改原请求，而是**创建一个新请求对象**（不可变设计）。原请求不变，新请求多了 `user-info` 头。

## 六、全链路用户信息传递

```
用户请求 → Gateway（解析Token，塞user-info到请求头）
              ↓
         trade-service（拦截器取user-info → 存UserContext）
              ↓
         trade-service用Feign调item-service
              ↓
         Feign拦截器（从UserContext取userId → 塞到Feign请求头）
              ↓
         item-service（同样取user-info → 存UserContext）
```

**核心思想：Token 只在网关解析一次，之后全链路用请求头传 userId。**

## 七、踩坑避坑指南

### 报错坑：spring-security-rsa 版本不兼容

- 报错信息：`Invalid keystore format`
- 原因：gateway 的 pom.xml 缺少 `spring-security-crypto` 和 `spring-security-rsa` 依赖，传递依赖拉到旧版本 `1.0.10.RELEASE`，跟项目用的 `1.0.9.RELEASE` 不兼容
- 解决方案：显式声明依赖，版本与 user-service 一致

## 八、效果验证

| 请求 | 状态码 | 说明 |
|---|---|---|
| `GET /items/1` | 200 | 白名单放行 |
| `GET /orders` | 401 | 非白名单，没 Token，被拦截 |
| `GET /carts` | 401 | 非白名单，没 Token，被拦截 |

## 九、全文总结

- GlobalFilter 是 Gateway 的全局过滤器，所有请求都会经过
- `chain.filter()` = 放行，不调用 = 拦截
- 白名单用 `AntPathMatcher` 做路径匹配
- 用户信息通过请求头 `user-info` 传递，下游用拦截器 + ThreadLocal 接收
- `request.mutate()` 创建新请求，不修改原请求（不可变设计）

## 十、关联笔记

- [[Gateway入门与路由配置]]
- [[OpenFeign快速入门]]
- [[Spring依赖注入与反射]]

## 十一、代码变更

- 新增：`hm-gateway/src/main/java/com/hmall/gateway/config/AuthProperties.java` — 白名单配置类
- 新增：`hm-gateway/src/main/java/com/hmall/gateway/config/JwtProperties.java` — JWT 配置类
- 新增：`hm-gateway/src/main/java/com/hmall/gateway/config/SecurityConfig.java` — 秘钥 Bean 配置
- 新增：`hm-gateway/src/main/java/com/hmall/gateway/filter/AuthGlobalFilter.java` — 鉴权过滤器
- 新增：`hm-gateway/src/main/java/com/hmall/gateway/util/JwtTool.java` — JWT 工具类
- 修改：`hm-gateway/pom.xml` — 添加 spring-security-crypto 和 spring-security-rsa 依赖
- 修改：`hm-gateway/src/main/resources/application.yaml` — 添加 hm.jwt 和 hm.auth 配置
