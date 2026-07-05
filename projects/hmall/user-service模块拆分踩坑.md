---
date: 2026-07-05
tags: [hmall, SpringCloud, 微服务, Nacos, SpringSecurity]
---

# 【微服务拆分】user-service 模块踩坑实录：配置遗漏与依赖缺失

## 一、写在前面

从单体架构拆分微服务时，每个独立模块都需要完整配置。稍有遗漏就会启动失败或功能异常。

**学完这篇你能掌握：**
- 微服务模块拆分后的配置检查清单
- Nacos 注册、Spring Security 依赖、包扫描三个高频踩坑点

---

## 二、前置准备

- 已完成 user-service 模块的基本拆分（Controller/Service/Mapper/Config）
- MySQL 运行中，hm-user 数据库已创建
- Nacos 运行中（192.168.188.130:8848）

---

## 三、正文：检查过程与问题发现

### 3.1 配置文件对比检查

将 user-service 与 item-service、cart-service 的 `application.yaml` 进行对比：

```yaml
# user-service 有问题的配置
spring:
  cloud:
    nacos:
      server-addr: 192.168.188.130  # ❌ 缺少端口号

# 正确配置（item-service、cart-service 都是这样）
spring:
  cloud:
    nacos:
      server-addr: 192.168.188.130:8848  # ✅
```

**坑点：** Nacos 默认端口是 8848，不写端口号会导致注册失败，但启动时可能不报错（只是注册不上）。

---

### 3.2 依赖检查

检查 pom.xml 发现缺少 Spring Security 依赖，但代码中使用了 `PasswordEncoder`：

```java
// UserServiceImpl.java 中使用了 PasswordEncoder
private final PasswordEncoder passwordEncoder;

// SecurityConfig.java 中创建了 BCryptPasswordEncoder
@Bean
public PasswordEncoder passwordEncoder(){
    return new BCryptPasswordEncoder();
}
```

**需要添加的依赖：**
```xml
<!-- hm-service 的 pom.xml 中有这两个依赖 -->
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

**坑点：** 这两个依赖在 hm-service 中有，拆分时漏掉了。`spring-security-crypto` 提供 `BCryptPasswordEncoder`，`spring-security-rsa` 提供 JWT 密钥对生成。

---

### 3.3 包扫描检查

检查启动类发现只扫描了 `com.hmall.user` 包：

```java
// 原来的配置
@SpringBootApplication  // 只扫描 com.hmall.user

// 需要改为
@SpringBootApplication(scanBasePackages = "com.hmall")  // 扫描整个 com.hmall 包
```

**为什么需要？** `hm-common` 模块中的 `CommonExceptionAdvice`（全局异常处理）、`UserContext`（用户上下文）等组件都在 `com.hmall.common` 包下，不扫描就无法生效。

**坑点：** `@SpringBootApplication` 默认只扫描启动类所在的包及其子包，跨模块的组件需要手动指定扫描范围。

---

## 四、方案对比

| 检查方式 | 优点 | 缺点 |
|---|---|---|
| **对比法**（对比已拆分成功的模块） | 快速发现遗漏 | 依赖有一个模块是正确的 |
| **启动测试法** | 直接看到报错 | 可能只暴露第一个问题 |
| **逐项清单法** | 不会遗漏 | 耗时较长 |

**推荐：** 对比法 + 启动测试，先对比配置文件，再启动验证。

---

## 五、踩坑避坑指南

### 逻辑坑（不报错但会出问题）

**坑 1：Nacos 地址漏端口号**
- 问题描述：只写 IP 不写端口号
- 后果：服务启动正常，但注册不到 Nacos，其他服务无法发现
- 正确做法：始终写 `IP:8848` 格式

**坑 2：包扫描范围不足**
- 问题描述：`@SpringBootApplication` 默认只扫描当前模块的包
- 后果：`CommonExceptionAdvice` 等公共组件不生效，异常无法统一处理
- 正确做法：添加 `scanBasePackages = "com.hmall"`

### 依赖坑

**坑 3：Spring Security 依赖遗漏**
- 报错信息：`NoClassDefFoundError: org/springframework/security/crypto/password/PasswordEncoder`
- 原因分析：拆分时只复制了代码，没有复制依赖
- 解决方案：从 hm-service 的 pom.xml 中复制对应的依赖

---

## 六、效果验证

修复后启动 user-service，访问登录接口：

```bash
POST http://localhost:8084/users/login
Content-Type: application/json

{
  "username": "Jack",
  "password": "123456"
}
```

**成功返回：**
```json
{
  "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9...",
  "userId": "1",
  "username": "Jack",
  "balance": 1000000
}
```

---

## 七、全文总结

**拆分微服务时的配置检查清单：**

1. ✅ **端口配置** — 每个服务独立端口
2. ✅ **服务名** — `spring.application.name` 唯一
3. ✅ **Nacos 地址** — 必须带端口号 `:8848`
4. ✅ **数据源** — 数据库名改为对应的库（如 hm-user）
5. ✅ **MyBatis-Plus 配置** — 枚举处理器、更新策略等
6. ✅ **日志配置** — 输出路径、日志级别
7. ✅ **Knife4j 配置** — api-rule-resources 指向当前模块的 controller 包
8. ✅ **依赖检查** — 对比原模块 pom.xml，确保依赖完整
9. ✅ **包扫描** — 确保扫描到 hm-common 的组件

---

## 八、关联笔记

- [[微服务拆分原则]]
- [[OpenFeign远程调用]]
- [[Nacos服务注册发现]]
- [[JWT认证机制]]

## 九、代码变更

- 修改：`user-service/src/main/resources/application.yaml` — Nacos 地址加端口号
- 修改：`user-service/pom.xml` — 添加 Spring Security 依赖
- 修改：`user-service/src/main/java/com/hmall/user/UserApplication.java` — 添加 scanBasePackages
