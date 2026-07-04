---
date: 2026-07-04
tags: [黑马商城微服务, SpringCloud, Nacos, 服务注册发现]
---

# 服务注册发现与 Nacos

## 一、写在前面

上一篇用 RestTemplate 实现了远程调用，但 URL 写死了 `localhost:8081`。问题来了：

- item-service 多实例部署，地址各不相同，cart-service 怎么知道？
- 某个实例宕机了，cart-service 还在调怎么办？
- 临时加了新实例，cart-service 怎么发现？

**注册中心**就是解决这些问题的。

## 二、注册中心原理

三个角色：**注册中心**、**服务提供者**（item-service）、**服务消费者**（cart-service）。

### 2.1 流程

```
服务提供者（item-service）          注册中心（Nacos）         服务消费者（cart-service）
      │                              │                              │
      │  ① 启动时注册自己的信息       │                              │
      │     （服务名、IP、端口）       │                              │
      ├────────────────────────────→│                              │
      │                              │                              │
      │                              │  ② 消费者订阅想要的服务       │
      │                              │←─────────────────────────────┤
      │                              │                              │
      │                              │  ③ 返回实例列表               │
      │                              ├─────────────────────────────→│
      │                              │                              │
      │                              │                  ④ 负载均衡选一个实例
      │                              │                              │
      │  ⑤ 消费者向选中的实例发起调用  │                              │
      │←───────────────────────────────────────────────────────────┤
      │                              │                              │
```

### 2.2 心跳机制（实例如何被发现/剔除）

```
时间线：
  0s    item-service-1 启动 → 注册到 Nacos
  0s    item-service-2 启动 → 注册到 Nacos
  30s   item-service-1 发心跳："我还活着"
  30s   item-service-2 发心跳："我还活着"
  60s   item-service-1 发心跳 ✓
  60s   item-service-2 没心跳（宕机了）
  90s   item-service-2 还是没心跳
        Nacos 判定：item-service-2 挂了 → 从实例列表剔除
        通知 cart-service：服务列表更新了
```

**关键机制**：
- 服务定期向注册中心发心跳（默认 5 秒一次）
- 注册中心长时间收不到心跳 → 剔除实例
- 新实例启动 → 自动注册
- 服务列表变更 → 主动通知消费者更新

### 2.3 通俗类比：外卖平台

| 概念 | 类比 |
|---|---|
| 注册中心 | 外卖平台（美团） |
| 服务提供者 | 商家（注册到平台） |
| 服务消费者 | 顾客（在平台点餐） |
| 服务名 | "黄焖鸡"（顾客只关心菜名） |
| 实例列表 | 平台显示有 3 家黄焖鸡店 |
| 负载均衡 | 顾客自己挑一家下单 |
| 心跳 | 商家每 30 秒报平安 |
| 实例下线 | 商家关门，平台自动下架 |

## 三、Nacos 配置

### 3.1 添加依赖（pom.xml）

```xml
<!--nacos 服务注册发现-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

> 版本由父 pom 的 `spring-cloud-alibaba-dependencies` 统一管理，不用写版本号。

### 3.2 配置 Nacos 地址（application.yaml）

```yaml
spring:
  application:
    name: item-service  # 服务名称（注册到 Nacos 的名字）
  cloud:
    nacos:
      server-addr: 192.168.188.130:8848  # Nacos 地址
```

### 3.3 启动验证

1. 启动 Nacos（虚拟机 `192.168.188.130:8848`）
2. 启动 item-service
3. 打开 Nacos 控制台 → 服务管理 → 服务列表
4. 看到 `item-service` 且状态为健康 = 注册成功

## 四、引入 Nacos 后的变化

```java
// 之前：写死地址
restTemplate.exchange("http://localhost:8081/items?ids={ids}", ...)

// 之后：只写服务名，Nacos 帮你解析到具体地址
restTemplate.exchange("http://item-service/items?ids={ids}", ...)
```

> 还需要给 RestTemplate 加 `@LoadBalanced` 注解才能用服务名，后面 OpenFeign 笔记会讲。

## 五、服务发现实操（cart-service）

### 5.1 cart-service 注册到 Nacos

**pom.xml 添加依赖：**

```xml
<!--nacos 服务注册发现-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<!--负载均衡-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

**application.yaml 配置：**

```yaml
spring:
  application:
    name: cart-service
  cloud:
    nacos:
      server-addr: 192.168.188.130:8848
```

### 5.2 服务发现 + 随机负载均衡

```java
@Service
@RequiredArgsConstructor
public class CartServiceImpl extends ServiceImpl<CartMapper, Cart> implements ICartService {

    private final RestTemplate restTemplate;
    private final DiscoveryClient discoveryClient;  // 注入 DiscoveryClient

    private void handleCartItems(List<CartVO> vos) {
        Set<Long> itemIds = vos.stream().map(CartVO::getItemId).collect(Collectors.toSet());

        // 1. 从 Nacos 获取 item-service 实例列表
        List<ServiceInstance> instances = discoveryClient.getInstances("item-service");
        if (CollUtils.isEmpty(instances)) {
            return;
        }

        // 2. 随机负载均衡，选一个实例
        ServiceInstance instance = instances.get(ThreadLocalRandom.current().nextInt(instances.size()));

        // 3. 用选中的实例地址发起调用
        ResponseEntity<List<ItemDTO>> response = restTemplate.exchange(
                instance.getUri() + "/items?ids={ids}",
                HttpMethod.GET,
                null,
                new ParameterizedTypeReference<List<ItemDTO>>() {},
                Map.of("ids", CollUtil.join(itemIds, ","))
        );
        // ... 解析响应
    }
}
```

**关键变化：**

| 之前（硬编码） | 之后（服务发现） |
|---|---|
| `http://localhost:8081/items` | `instance.getUri() + "/items"` |
| 地址写死，多实例没法用 | 自动发现所有实例，负载均衡 |

### 5.3 踩坑记录

**坑 1：Nacos 连不上**
- 报错：`Connection refused: /192.168.188.130:9848`
- 原因：Nacos 2.x 用 gRPC，端口是 8848+1000=9848，需要确保虚拟机 Nacos 容器正常运行
- 解决：`docker ps | grep nacos` 检查容器状态

**坑 2：端口被占用**
- 报错：`Port 8082 was already in use`
- 原因：QQ.exe 随机占了 8082 端口（离谱）
- 解决：`netstat -ano | grep 8082` 找 PID，`taskkill /PID xxx /F` 杀掉

## 六、全文总结

1. **注册中心解决什么**：服务发现、实例管理、故障剔除、动态扩缩容
2. **核心流程**：服务注册 → 消费者订阅 → 获取实例列表 → 负载均衡 → 发起调用
3. **心跳机制**：服务定期报告健康状态，长时间不响应则剔除
4. **Nacos 配置**：加依赖 + 配 `server-addr` + 配 `spring.application.name`

## 六、关联笔记

- [[远程调用与RestTemplate]] — 前置知识，RestTemplate 用法
- [[微服务拆分原则与实操]] — 拆分的理论和实操
