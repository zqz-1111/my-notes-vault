---
date: 2026-07-05
tags: [黑马商城微服务, SpringCloud, OpenFeign, 微服务拆分]
---

# 【OpenFeign】trade-service 与 pay-service 抽取实战

## 一、写在前面

单体应用拆成微服务后，各服务在不同 JVM 进程里运行，不能再用 `this.xxx()` 直接调用。本文记录如何用 **OpenFeign** 完成 trade-service（交易服务）和 pay-service（支付服务）的远程调用改造。

学完这篇你能掌握：
- 如何从单体中抽取业务服务（trade/pay）
- 如何在 hm-api 中定义 FeignClient 接口
- 如何将本地调用改造为 Feign 远程调用
- 支付流程的跨服务调用链路

## 二、前置准备

- 已完成 item-service、cart-service、user-service 的拆分（见 [[item-service与cart-service模块拆分]]）
- hm-api 模块已存在，包含 `ItemClient`、`ItemDTO`、`DefaultFeignConfig`
- Nacos 注册中心运行中

## 三、trade-service 抽取

### 3.1 模块结构

```
trade-service/
├── pom.xml                          ← 依赖 hm-common + hm-api
└── src/main/
    ├── java/com/hmall/trade/
    │   ├── TradeApplication.java    ← @EnableFeignClients(basePackages = "com.hmall.api.client")
    │   ├── controller/OrderController.java
    │   ├── service/impl/OrderServiceImpl.java
    │   ├── mapper/                  ← OrderMapper, OrderDetailMapper, OrderLogisticsMapper
    │   └── domain/                  ← PO/DTO/VO
    └── resources/
        ├── application.yaml         ← 端口 8085，数据库 hm-trade
        └── application-dev/local.yaml
```

### 3.2 抽取 FeignClient

下单时 trade-service 需要调用商品服务（查商品、扣库存）和购物车服务（清购物车），需要在 hm-api 中定义对应的 Feign 客户端。

#### ① 补全 ItemClient —— 加 deductStock 方法

原来 hm-api 的 `ItemClient` 只有 `queryItemByIds`，需要加上扣减库存接口：

```java
@FeignClient("item-service")
public interface ItemClient {
    @GetMapping("/items")
    List<ItemDTO> queryItemByIds(@RequestParam("ids") Collection<Long> ids);

    // 新增：扣减库存
    @PutMapping("/items/stock/deduct")
    void deductStock(@RequestBody List<OrderDetailDTO> items);
}
```

#### ② 抽取共享 DTO —— OrderDetailDTO

`deductStock` 方法的参数 `OrderDetailDTO`（itemId + num）在 item-service 和 trade-service 都要用，放到 hm-api 的 `com.hmall.api.dto` 包下共享：

```java
@ApiModel(description = "订单明细条目")
@Data
@Accessors(chain = true)
public class OrderDetailDTO {
    @ApiModelProperty("商品id")
    private Long itemId;
    @ApiModelProperty("商品购买数量")
    private Integer num;
}
```

> **踩坑**：item-service 和 trade-service 各自有一份 `OrderDetailDTO`，包名不同。Feign 序列化靠字段名匹配所以能跑通，但维护两份容易出错，统一放到 hm-api 更干净。

#### ③ 新建 CartClient

购物车服务的批量删除接口：`DELETE /carts?ids=...`

```java
@FeignClient("cart-service")
public interface CartClient {
    @DeleteMapping("/carts")
    void deleteCartItemByIds(@RequestParam("ids") Collection<Long> ids);
}
```

### 3.3 改造 OrderServiceImpl

**改造前**（本地调用，编译不过）：

```java
private final IItemService itemService;   // 不存在于 trade-service
private final ICartService cartService;   // 不存在于 trade-service
```

**改造后**（Feign 远程调用）：

```java
private final ItemClient itemClient;      // hm-api 中的 Feign 客户端
private final CartClient cartClient;      // hm-api 中的 Feign 客户端
```

方法调用从 `itemService.xxx()` 改成 `itemClient.xxx()`，代码写法几乎不变，但底层从 JVM 内部调用变成了 HTTP 远程调用。

### 3.4 下单流程调用链

```
用户 → trade-service:8085 POST /orders
        │
        ├─ ① itemClient.queryItemByIds()     → item-service:8083 GET /items
        ├─ ② 本地计算总价，保存订单到 hm-trade 库
        ├─ ③ itemClient.deductStock()         → item-service:8083 PUT /items/stock/deduct
        └─ ④ cartClient.deleteCartItemByIds() → cart-service:8084 DELETE /carts
```

## 四、pay-service 抽取

### 4.1 模块结构

```
pay-service/
├── pom.xml
└── src/main/java/com/hmall/pay/
    ├── PayApplication.java
    ├── controller/PayController.java
    ├── service/impl/PayOrderServiceImpl.java
    ├── mapper/PayOrderMapper.java
    ├── domain/          ← PayOrder, PayApplyDTO, PayOrderFormDTO, PayOrderVO
    └── enums/           ← PayStatus, PayChannel, PayType
```

端口 **8086**，数据库 **hm-pay**。

### 4.2 抽取 FeignClient

支付时需要扣用户余额（user-service）和标记订单已支付（trade-service）。

#### ① 新建 UserClient

```java
@FeignClient("user-service")
public interface UserClient {
    @PutMapping("/users/money/deduct")
    void deductMoney(@RequestParam("pw") String pw, @RequestParam("amount") Integer amount);
}
```

对应 user-service 的 `PUT /users/money/deduct?pw=xx&amount=xx` 接口。

#### ② 新建 TradeClient

```java
@FeignClient("trade-service")
public interface TradeClient {
    @PutMapping("/orders/{orderId}")
    void markOrderPaySuccess(@PathVariable("orderId") Long orderId);
}
```

对应 trade-service 的 `PUT /orders/{orderId}` 接口。

> **注意**：之前把 trade-service 的 `markOrderPaySuccess` 端点删了，现在 pay-service 要调用它，必须加回来。

### 4.3 改造 PayOrderServiceImpl

**改造前**（本地调用，编译不过）：

```java
private final IUserService userService;    // 不存在于 pay-service
private final IOrderService orderService;  // 不存在于 pay-service

// 支付成功后手动构建 Order 对象更新状态
Order order = new Order();
order.setId(po.getBizOrderNo());
order.setStatus(2);
order.setPayTime(LocalDateTime.now());
orderService.updateById(order);
```

**改造后**（Feign 远程调用）：

```java
private final UserClient userClient;       // hm-api 中的 Feign 客户端
private final TradeClient tradeClient;     // hm-api 中的 Feign 客户端

// 支付成功后直接调 trade-service 的接口
tradeClient.markOrderPaySuccess(po.getBizOrderNo());
```

改造后更简洁——订单状态更新的逻辑由 trade-service 自己负责，pay-service 只负责"告诉"它订单已支付。

### 4.4 支付流程调用链

```
用户 → pay-service:8086 POST /pay-orders/{id}
        │
        ├─ ① 本地查询支付单，校验状态（幂等性）
        ├─ ② userClient.deductMoney()         → user-service:8082 PUT /users/money/deduct
        ├─ ③ 本地更新支付单状态为已支付
        └─ ④ tradeClient.markOrderPaySuccess() → trade-service:8085 PUT /orders/{orderId}
```

## 五、hm-api 最终结构

```
hm-api（jar 包，不独立运行，被所有服务依赖）
├── client/
│   ├── ItemClient.java      ← @FeignClient("item-service")
│   ├── CartClient.java      ← @FeignClient("cart-service")
│   ├── UserClient.java      ← @FeignClient("user-service")
│   └── TradeClient.java     ← @FeignClient("trade-service")
├── dto/
│   ├── ItemDTO.java
│   └── OrderDetailDTO.java
└── config/
    └── DefaultFeignConfig.java
```

## 六、方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|
| 本地调用（单体） | 无网络开销，事务可控 | 耦合度高，无法独立扩缩容 | 小项目、快速验证 |
| RestTemplate | 简单直观 | URL 硬编码，无服务发现 | 简单调用、学习用 |
| OpenFeign | 声明式、自动服务发现、可读性好 | 多一次 HTTP 开销 | **微服务间调用（推荐）** |

选择 OpenFeign 的原因：接口+注解的声明式写法最接近本地调用的体验，配合 Nacos 自动服务发现，不需要手动管理 URL。

## 七、踩坑避坑指南

### 逻辑坑

**坑 1：删了被依赖的接口**
- 问题：之前把 trade-service 的 `markOrderPaySuccess` 删了，后来 pay-service 需要调用它
- 后果：pay-service 的 `TradeClient` 调用 404
- 教训：删接口前先检查有没有其他服务依赖它

**坑 2：DTO 不共享导致维护两份**
- 问题：item-service 和 trade-service 各自有一份 `OrderDetailDTO`
- 后果：改了一边忘了另一边，字段不一致
- 正确做法：统一放到 hm-api 共享

### 报错坑

**坑 3：pay-service 的 IUserService 找不到**
- 报错：`NoSuchBeanDefinitionException: No qualifying bean of type 'IUserService'`
- 原因：从 hm-service 复制代码时没有改依赖注入
- 解决：`IUserService` → `UserClient`，`IOrderService` → `TradeClient`

## 八、效果验证

1. 启动 Nacos + 5 个微服务
2. 调用 `POST /orders` 创建订单 → 返回订单 ID，总价计算正确
3. 调用 `POST /pay-orders` 生成支付单 → 返回支付单 ID
4. 调用 `POST /pay-orders/{id}` 余额支付 → 支付单状态变 3（TRADE_SUCCESS），订单状态变 2（已支付）
5. 调用 `GET /pay-orders` 查询支付单 → 确认状态正确

## 九、全文总结

- **trade-service 抽取**：OrderServiceImpl 中 `IItemService` → `ItemClient`，`ICartService` → `CartClient`
- **pay-service 抽取**：PayOrderServiceImpl 中 `IUserService` → `UserClient`，`IOrderService` → `TradeClient`
- **共享 DTO**：`OrderDetailDTO` 从各服务本地移到 hm-api
- **核心思路**：复制代码 → 改包名 → 本地调用改 Feign 调用 → 配独立数据库和端口 → 注册到 Nacos
- **hm-api 的角色**：不是独立服务，是所有服务共享的 Feign 客户端 + DTO jar 包

## 十、关联笔记

- [[OpenFeign入门与hm-api公共模块]]
- [[item-service与cart-service模块拆分]]
- [[微服务拆分原则与架构总览]]

## 十一、代码变更

### hm-api 模块
- 新增：`CartClient.java` — 购物车批量删除 Feign 客户端
- 新增：`UserClient.java` — 用户扣余额 Feign 客户端
- 新增：`TradeClient.java` — 订单标记已支付 Feign 客户端
- 新增：`OrderDetailDTO.java` — 共享 DTO
- 修改：`ItemClient.java` — 新增 deductStock 方法

### item-service 模块
- 修改：`pom.xml` — 添加 hm-api 依赖
- 修改：`ItemController.java` — import 改用 hm-api 的 OrderDetailDTO
- 修改：`IItemService.java` — 同上
- 修改：`ItemServiceImpl.java` — 同上
- 删除：`OrderDetailDTO.java` — 用 hm-api 的替代

### trade-service 模块
- 修改：`OrderServiceImpl.java` — IItemService→ItemClient，ICartService→CartClient，加回 markOrderPaySuccess
- 修改：`IOrderService.java` — 加回 markOrderPaySuccess 声明
- 修改：`OrderController.java` — 加回 PUT /{orderId} 端点
- 修改：`OrderFormDTO.java` — import 改用 hm-api 的 OrderDetailDTO
- 删除：`OrderDetailDTO.java` — 用 hm-api 的替代

### pay-service 模块（新建）
- 新增：全部文件（16 个 Java 文件 + 3 个 YAML 配置）
- 修改：`PayOrderServiceImpl.java` — IUserService→UserClient，IOrderService→TradeClient
- 修改：`PayController.java` — 新增 GET /pay-orders 测试接口
