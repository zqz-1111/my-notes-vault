---
date: 2026-07-06
tags: [黑马商城微服务, SpringCloud, Nacos, 配置管理]
---

# 【Nacos】统一配置管理与热更新

## 一、写在前面

### 1.1 解决什么问题？

到目前为止，微服务已经解决了远程调用、注册发现、路由负载均衡、用户信息传递等问题。但还有几个痛点：

1. **网关路由配置写死**：改配置必须重启微服务
2. **业务配置写死**：每次修改都要重启服务
3. **重复配置多**：每个微服务都有数据库、日志、Swagger 等相同配置，维护成本高

### 1.2 学完这篇你能掌握什么？

- Nacos 配置管理的核心概念
- 如何提取共享配置到 Nacos
- 如何实现配置热更新（不用重启）

## 二、前置准备

- Nacos 已安装并启动
- 微服务已注册到 Nacos
- 了解 Spring Boot 配置文件结构

## 三、核心概念

### 3.1 Nacos 配置模型

Nacos 使用 **Data ID + Group + Namespace** 三元组定位配置：

| 概念 | 说明 | 示例 |
|------|------|------|
| Data ID | 配置文件名 | `shared-jdbc.yaml` |
| Group | 配置分组 | `SHARED_GROUP` |
| Namespace | 命名空间（环境隔离） | `dev`、`test`、`prod` |

### 3.2 启动顺序

Spring Cloud 的配置加载分两个阶段：

```
引导阶段 → SpringBoot阶段
```

1. **引导阶段**：读取 `bootstrap.yaml` → 连接 Nacos → 拉取配置
2. **SpringBoot 阶段**：合并配置 → 初始化应用

关键点：`application.yaml` 在 SpringBoot 阶段才读取，所以引导阶段必须用 `bootstrap.yaml` 配置 Nacos 地址。

## 四、实操步骤

### 4.1 引入依赖

在需要配置管理的微服务中添加：

```xml
<!--nacos配置管理-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
<!--读取bootstrap文件-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```

**说明**：
- `spring-cloud-starter-alibaba-nacos-config`：Nacos 配置管理功能
- `spring-cloud-starter-bootstrap`：让 Spring 识别 `bootstrap.yaml`

### 4.2 创建 bootstrap.yaml

```yaml
spring:
  application:
    name: cart-service # 服务名称
  profiles:
    active: dev
  cloud:
    nacos:
      server-addr: 192.168.188.130:8848 # nacos地址
      config:
        file-extension: yaml # 文件后缀名
        shared-configs: # 共享配置
          - dataId: shared-jdbc.yaml # 共享mybatis配置
          - dataId: shared-log.yaml # 共享日志配置
          - dataId: shared-swagger.yaml # 共享swagger配置
```

**说明**：
- `server-addr`：Nacos 服务器地址
- `file-extension`：配置文件后缀
- `shared-configs`：要拉取的共享配置列表

### 4.3 精简 application.yaml

将共享配置移到 Nacos 后，`application.yaml` 只保留本服务特有配置：

```yaml
server:
  port: 8082
feign:
  okhttp:
    enabled: true # 开启OKHttp连接池支持
hm:
  swagger:
    title: 购物车服务接口文档
    package: com.hmall.cart.controller
  db:
    database: hm-cart
  cart:
    maxAmount: 10 # 购物车商品上限
```

**对比**：
- 删除：`spring.application.name`、`spring.profiles.active`、`spring.cloud.nacos` → 移到 `bootstrap.yaml`
- 删除：`spring.datasource.*`、`mybatis-plus.*` → 移到 `shared-jdbc.yaml`
- 删除：`logging.*` → 移到 `shared-log.yaml`
- 删除：`knife4j.*` → 移到 `shared-swagger.yaml`
- 保留：端口、数据库名、服务特有配置

### 4.4 在 Nacos 控制台创建共享配置

#### shared-jdbc.yaml

```yaml
spring:
  datasource:
    url: jdbc:mysql://${hm.db.host:192.168.150.101}:${hm.db.port:3306}/${hm.db.database}?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: ${hm.db.un:root}
    password: ${hm.db.pw:123}
mybatis-plus:
  configuration:
    default-enum-type-handler: com.baomidou.mybatisplus.core.handlers.MybatisEnumTypeHandler
  global-config:
    db-config:
      update-strategy: not_null
      id-type: auto
```

#### shared-log.yaml

```yaml
logging:
  level:
    com.hmall: debug
  pattern:
    dateformat: HH:mm:ss:SSS
  file:
    path: "logs/${spring.application.name}"
```

#### shared-swagger.yaml

```yaml
knife4j:
  enable: true
  openapi:
    title: ${hm.swagger.title:黑马商城接口文档}
    description: ${hm.swagger.description:黑马商城接口文档}
    email: ${hm.swagger.email:zhanghuyi@itcast.cn}
    concat: ${hm.swagger.concat:虎哥}
    url: https://www.itcast.cn
    version: v1.0.0
    group:
      default:
        group-name: default
        api-rule: package
        api-rule-resources:
          - ${hm.swagger.package}
```

**注意**：创建时 Group 填 `SHARED_GROUP`，格式选 `YAML`。

### 4.5 配置热更新

#### 创建属性读取类

```java
package com.hmall.cart.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Data
@Component
@ConfigurationProperties(prefix = "hm.cart")
public class CartProperties {
    private Integer maxAmount;
}
```

**说明**：
- `@ConfigurationProperties(prefix = "hm.cart")`：绑定 `hm.cart` 前缀的配置
- `@Component`：注册为 Spring Bean
- `@Data`：Lombok 自动生成 getter/setter

#### 在业务中使用

```java
@Service
@RequiredArgsConstructor
public class CartServiceImpl extends ServiceImpl<CartMapper, Cart> implements ICartService {

    private final CartProperties cartProperties;

    private void checkCartsFull(Long userId) {
        int count = lambdaQuery().eq(Cart::getUserId, userId).count();
        if (count >= cartProperties.getMaxAmount()) {
            throw new BizIllegalException(StrUtil.format("用户购物车课程不能超过{}", cartProperties.getMaxAmount()));
        }
    }
}
```

**热更新原理**：
- `@ConfigurationProperties` + `@Component` 组合
- Nacos 配置变更时，Spring 自动刷新 Bean 属性
- 不用重启服务即可生效

## 五、踩坑避坑指南

### 5.1 逻辑坑

**坑 1：bootstrap.yaml 不生效**
- 问题：Spring 不识别 `bootstrap.yaml`
- 原因：缺少 `spring-cloud-starter-bootstrap` 依赖
- 解决：添加依赖

**坑 2：共享配置拉取不到**
- 问题：启动报错找不到配置
- 原因：`bootstrap.yaml` 中 `dataId` 写错或 Group 不匹配
- 解决：检查 Nacos 控制台的 Data ID 和 Group

### 5.2 报错坑

**坑 3：Nacos 连接失败**
- 报错：`Connect to Nacos server failed`
- 原因：地址或端口错误
- 解决：检查 `server-addr` 配置，确保 Nacos 服务正常

## 六、效果验证

1. 重启服务，看日志是否显示拉取到共享配置
2. 访问 `http://localhost:8082/doc.html`，Swagger 文档正常说明 `shared-swagger.yaml` 生效
3. 测试热更新：在 Nacos 修改 `hm.cart.maxAmount`，不重启直接测试

## 七、全文总结

- **Nacos 配置管理**：统一管理微服务配置，支持动态刷新
- **bootstrap.yaml**：引导阶段读取，配置 Nacos 地址
- **共享配置**：提取公共配置（数据库、日志、Swagger）到 Nacos
- **热更新**：`@ConfigurationProperties` + `@Component` 实现配置变更自动刷新

## 八、关联笔记

- [[Gateway网关与路由配置]]
- [[OpenFeign远程调用]]
- [[微服务拆分原则]]

## 九、代码变更

- 新增：`cart-service/src/main/resources/bootstrap.yaml` — Nacos 配置
- 修改：`cart-service/src/main/resources/application.yaml` — 精简配置
- 新增：`cart-service/src/main/java/com/hmall/cart/config/CartProperties.java` — 属性读取类
- 修改：`cart-service/src/main/java/com/hmall/cart/service/impl/CartServiceImpl.java` — 使用 CartProperties
- 修改：`cart-service/pom.xml` — 添加 Nacos 配置依赖
