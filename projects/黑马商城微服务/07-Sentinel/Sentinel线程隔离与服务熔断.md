---
date: 2026-07-08
tags: [hmall, SpringCloud, Sentinel, 微服务]
---

# 【Sentinel】线程隔离与服务熔断

## 一、写在前面

1. 为什么学这个：限流能降低服务器压力，但不能完全避免服务故障。一旦某个服务出问题，需要隔离调用，避免雪崩
2. 学完这篇你能掌握：OpenFeign 整合 Sentinel、FallbackFactory 降级逻辑、断路器熔断机制

---

## 二、线程隔离

### 2.1 为什么需要线程隔离

查询购物车时需要调用商品服务。如果商品服务故障，会导致购物车服务级联失败。

**解决方案**：把查询商品的部分隔离起来，限制可用线程资源。即使商品服务挂了，最多导致查询购物车故障，不会拖垮整个购物车服务。

### 2.2 OpenFeign 整合 Sentinel

修改 cart-service 的 `application.yaml`：

```yaml
feign:
  sentinel:
    enabled: true # 开启feign对sentinel的支持
```

### 2.3 Tomcat 线程配置（测试用）

默认 Tomcat 最大线程数 200、最大连接 8492，单机测试很难打满。测试时可以调小：

```yaml
server:
  port: 8082
  tomcat:
    threads:
      max: 50 # 允许的最大线程数
    accept-count: 50 # 最大排队等待数量
    max-connections: 100 # 允许的最大连接
```

---

## 三、QPS vs 并发线程数

| 指标 | 关注点 | 例子 |
|------|--------|------|
| QPS（每秒查询数） | 流量速度 | 每秒来了 100 个请求 |
| 并发线程数 | 资源占用 | 同时有 10 个请求在处理中 |

**通俗理解**：一个请求处理需要 1 秒，每秒来 10 个请求 → QPS = 10，并发线程数 = 10

**Sentinel 中怎么选**：
- QPS 限流：限制请求速度，保护接口不被打爆
- 并发线程数限流：限制资源占用，隔离慢调用（比如远程调用卡住）

---

## 四、服务熔断

### 4.1 为什么需要熔断

查询商品延迟高（模拟 500ms），导致查询购物车响应也变长。问题：
1. 超出 QPS 上限的请求只能报错，购物车查询失败
2. 拖慢购物车服务，消耗更多资源，用户体验差

**解决方案**：对于不健康的接口，直接停止调用，走降级逻辑。

### 4.2 编写降级逻辑（FallbackFactory）

给 FeignClient 编写降级逻辑有两种方式：
- **FallbackClass**：无法对远程调用的异常做处理
- **FallbackFactory**：可以对异常做处理，推荐使用

#### 步骤一：创建 FallbackFactory

```java
package com.hmall.api.client.fallback;

import com.hmall.api.client.ItemClient;
import com.hmall.api.dto.ItemDTO;
import com.hmall.api.dto.OrderDetailDTO;
import com.hmall.common.exception.BizIllegalException;
import com.hmall.common.utils.CollUtils;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.openfeign.FallbackFactory;

import java.util.Collection;
import java.util.List;

@Slf4j
public class ItemClientFallback implements FallbackFactory<ItemClient> {
    @Override
    public ItemClient create(Throwable cause) {
        return new ItemClient() {
            @Override
            public List<ItemDTO> queryItemByIds(Collection<Long> ids) {
                log.error("远程调用ItemClient#queryItemByIds方法出现异常，参数：{}", ids, cause);
                // 查询购物车允许失败，返回空集合
                return CollUtils.emptyList();
            }

            @Override
            public void deductStock(List<OrderDetailDTO> items) {
                // 库存扣减需要触发事务回滚，抛出异常
                throw new BizIllegalException(cause);
            }
        };
    }
}
```

#### 步骤二：注册为 Bean

在 `DefaultFeignConfig` 中添加：

```java
@Bean
public ItemClientFallback itemClientFallback(){
    return new ItemClientFallback();
}
```

#### 步骤三：绑定到 FeignClient

```java
@FeignClient(value = "item-service", fallbackFactory = ItemClientFallback.class)
public interface ItemClient {
    // ...
}
```

### 4.3 降级策略设计

| 方法 | 降级处理 | 原因 |
|------|----------|------|
| `queryItemByIds` | 返回空集合 | 查询失败不影响购物车展示，用户体验优先 |
| `deductStock` | 抛出异常 | 库存扣减失败必须回滚，不能静默处理 |

---

## 五、断路器状态机

### 5.1 三个状态

```
        超过阈值
closed ──────────→ open
   ↑                 │
   │  请求成功        │ 5秒后
   └──── half-open ←─┘
            │
            │ 请求失败
            ↓
          open
```

- **closed（关闭）**：正常状态，放行所有请求，统计异常比例/慢请求比例。超过阈值切换到 open
- **open（打开）**：熔断状态，拒绝所有请求，走降级逻辑。持续一段时间后进入 half-open
- **half-open（半开）**：放行一次请求试探
  - 成功 → 切换到 closed（恢复正常）
  - 失败 → 切换到 open（继续熔断）

### 5.2 通俗理解

就像电路断路器：
- 正常时（closed）：电流通着，但会检测异常
- 跳闸（open）：发现短路，直接断电保护
- 试探（half-open）：等一会儿，试一下电修好了没
  - 修好了 → 恢复供电
  - 还是短路 → 继续断电

---

## 六、踩坑避坑

### 踩坑 1：Sentinel 检测不到服务

- 问题：配置完 Sentinel 后 Dashboard 看不到 cart-service
- 原因：Sentinel 是懒加载的，必须先访问一次接口才会注册
- 解决：重启服务后，先访问一次接口（如 `GET /carts`），再去 Dashboard 查看

---

## 七、全文总结

- 线程隔离：限制某个服务调用的线程数，避免级联故障
- FallbackFactory：给 FeignClient 写降级逻辑，查询返回空集合，写操作抛异常
- 断路器：closed → open → half-open 三状态切换，自动熔断不健康接口

---

## 八、关联笔记

- [[Gateway鉴权过滤器实现]]
- [[OpenFeign快速入门]]
- [[动态路由]]

## 九、代码变更

- 新增：`hm-api/src/main/java/com/hmall/api/client/fallback/ItemClientFallback.java` — Feign 降级逻辑
- 修改：`hm-api/src/main/java/com/hmall/api/config/DefaultFeignConfig.java` — 注册 Fallback Bean
- 修改：`hm-api/src/main/java/com/hmall/api/client/ItemClient.java` — 绑定 fallbackFactory
- 修改：`cart-service/src/main/resources/application.yaml` — 开启 Sentinel + Tomcat 线程配置
