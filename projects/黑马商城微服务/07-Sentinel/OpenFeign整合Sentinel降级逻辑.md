---
date: 2026-07-09
tags:
  - 黑马商城微服务
  - Sentinel
  - OpenFeign
  - 降级
---

# 【Sentinel】OpenFeign整合Sentinel降级逻辑

## 一、写在前面

1. **为什么学这个？**
   - 微服务之间通过 OpenFeign 调用，如果被调用的服务挂了怎么办？
   - 没有降级逻辑 → 调用失败 → 用户看到500错误 → 体验极差
   - 有了降级逻辑 → 调用失败 → 返回兜底数据或友好提示 → 体验好

2. **学完你能掌握什么**
   - 给所有 FeignClient 编写 FallbackFactory 降级逻辑
   - 各微服务整合 Sentinel 的完整配置
   - 踩坑避坑指南

## 二、前置准备

- 已有 Sentinel Dashboard 部署
- 已有 OpenFeign 基础
- 了解 FallbackFactory 模式

## 三、正文

### 3.1 第一步：编写 FallbackFactory

FallbackFactory 是降级工厂，当 Feign 调用失败时，会调用这里的方法返回兜底数据。

**核心原则：**
| 方法类型 | 降级策略 | 原因 |
|---|---|---|
| 查询类方法 | 返回空集合/null | 不影响事务，允许失败 |
| 写操作方法 | 抛出异常 | 让 Seata 感知并回滚 |

**示例代码：**

```java
@Slf4j
public class UserClientFallback implements FallbackFactory<UserClient> {
    @Override
    public UserClient create(Throwable cause) {
        return new UserClient() {
            @Override
            public void deductMoney(String pw, Integer amount) {
                // 扣减余额是写操作，失败需要触发事务回滚
                log.error("远程调用UserClient#deductMoney方法出现异常，参数：pw={}, amount={}", pw, amount, cause);
                throw new BizIllegalException(cause);
            }
        };
    }
}
```

**关键点：**
- 实现 `FallbackFactory<T>` 接口
- `create` 方法返回目标接口的匿名实现
- 写操作必须抛异常，不能吞掉！

### 3.2 第二步：修改 @FeignClient 注解

在 FeignClient 接口上指定 `fallbackFactory`：

```java
@FeignClient(value = "user-service", fallbackFactory = UserClientFallback.class)
public interface UserClient {
    @PutMapping("/users/money/deduct")
    void deductMoney(@RequestParam("pw") String pw, @RequestParam("amount") Integer amount);
}
```

### 3.3 第三步：注册 FallbackFactory 为 Bean

**这一步很关键！** FallbackFactory 必须注册为 Spring Bean，否则 Sentinel 找不到。

在 `DefaultFeignConfig` 中添加：

```java
@Bean
public UserClientFallback userClientFallback(){
    return new UserClientFallback();
}

@Bean
public TradeClientFallback tradeClientFallback(){
    return new TradeClientFallback();
}
```

### 3.4 第四步：各微服务添加 Sentinel 配置

**pom.xml 添加依赖：**

```xml
<!--sentinel-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

**application.yaml 添加配置：**

```yaml
feign:
  sentinel:
    enabled: true # 开启feign对sentinel的支持
```

**注意：配置在 `feign:` 下面，不是 `spring:` 下面！**

## 四、完整修改清单

| FeignClient | FallbackFactory | 状态 |
|---|---|---|
| ItemClient | ItemClientFallback | ✅ 已有 |
| TradeClient | TradeClientFallback | ✅ 新增 |
| UserClient | UserClientFallback | ✅ 新增 |
| CartClient | 无方法，跳过 | ⏭️ |

| 微服务 | sentinel 依赖 | feign.sentinel.enabled |
|---|---|---|
| cart-service | ✅ 已有 | ✅ 已有 |
| trade-service | ✅ 新增 | ✅ 新增 |
| pay-service | ✅ 新增 | ✅ 新增 |
| user-service | ✅ 新增 | ✅ 新增 |

## 五、踩坑避坑指南

### 坑1：FallbackFactory 没有注册为 Bean

**问题：** 启动时报错 `No fallbackFactory instance of type class xxx found for feign client xxx`

**原因：** Sentinel 需要从 Spring 容器获取 FallbackFactory 实例

**解决：** 在 `DefaultFeignConfig` 中用 `@Bean` 注册

### 坑2：feign.sentinel.enabled 配置位置错误

**问题：** 配置了但不生效

**原因：** 配置放在了 `spring:` 下面，应该在 `feign:` 下面

**正确写法：**
```yaml
feign:
  sentinel:
    enabled: true
```

**错误写法：**
```yaml
spring:
  sentinel:
    enabled: true
```

### 坑3：local profile 覆盖主配置

**问题：** `application.yaml` 配置了，但 `application-local.yaml` 没有

**解决：** 各环境的 yaml 要保持一致，或者确保 local 会继承主配置

### 坑4：scanBasePackages 配置遗漏

**问题：** `MvcConfig` 没有生效，`UserInfoInterceptor` 不工作

**原因：** 启动类默认只扫描当前包，`com.hmall.common` 下的配置类需要显式扫描

**解决：**
```java
@SpringBootApplication(scanBasePackages = "com.hmall")
```

## 六、效果验证

### 测试1：正常调用

所有服务正常时，功能正常：
- 创建订单 ✅
- 创建支付单 ✅
- 支付成功 ✅

### 测试2：触发降级

停掉 `user-service`，再支付：
```
ERROR ... UserClientFallback : 远程调用UserClient#deductMoney方法出现异常，参数：pw=123, amount=144600

Caused by: BizIllegalException: [503] during [PUT] to [http://user-service/users/money/deduct] 
[Load balancer does not contain an instance for the service user-service]
```

**结果：** ✅ 降级触发，抛出异常，可被 Seata 捕获回滚

## 七、全文总结

1. **降级逻辑设计原则**：查询返回空，写操作抛异常
2. **FallbackFactory 必须注册为 Bean**：否则 Sentinel 找不到
3. **配置位置很重要**：`feign.sentinel.enabled` 在 `feign:` 下
4. **scanBasePackages 别忘了**：确保配置类被扫描到

## 八、关联笔记

- [[Sentinel线程隔离与服务熔断]]
- [[服务保护方案理论]]

## 九、代码变更

- 新增：`hm-api/.../fallback/TradeClientFallback.java` — TradeClient 降级逻辑
- 新增：`hm-api/.../fallback/UserClientFallback.java` — UserClient 降级逻辑
- 修改：`hm-api/.../client/TradeClient.java` — 添加 fallbackFactory 注解
- 修改：`hm-api/.../client/UserClient.java` — 添加 fallbackFactory 注解
- 修改：`hm-api/.../config/DefaultFeignConfig.java` — 注册 FallbackFactory Bean
- 修改：`trade-service/pom.xml` — 添加 sentinel 依赖
- 修改：`trade-service/application.yaml` — 添加 feign.sentinel.enabled
- 修改：`pay-service/pom.xml` — 添加 sentinel 依赖
- 修改：`pay-service/application.yaml` — 添加 feign.sentinel.enabled
- 修改：`pay-service/application-local.yaml` — 添加 feign.sentinel.enabled
- 修改：`user-service/pom.xml` — 添加 sentinel 依赖
- 修改：`user-service/application.yaml` — 添加 feign.sentinel.enabled
- 修改：`trade-service/TradeApplication.java` — 添加 scanBasePackages
- 修改：`pay-service/PayApplication.java` — 添加 scanBasePackages
- 新增：`trade-service/.../config/MvcConfig.java` — 解决用户信息传递问题
