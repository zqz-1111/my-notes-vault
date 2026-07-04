---
date: 2026-07-04
tags: [黑马商城微服务, SpringCloud, RestTemplate, 远程调用]
---

# 远程调用与 RestTemplate

## 一、写在前面

微服务拆分后，cart-service 和 item-service 在不同的 JVM 进程里，不能再直接 `new` 对象调方法了。怎么办？**通过网络发 HTTP 请求**——这就是远程调用。

## 二、本地调用 vs 远程调用

```
【单体时代】同一个 JVM，直接调
  CartServiceImpl
      ↓  直接注入，内存里调方法（纳秒级）
  ItemService.queryItemByIds(ids)

【微服务拆分后】不同 JVM，甚至不同机器
  cart-service (8082)          item-service (8081)
  CartServiceImpl ── HTTP ──→  /items?ids=xxx
      ↓ 发请求（毫秒级）           ↓ 查数据库
      ←── JSON 响应 ←──────── 返回结果
```

| | 本地调用 | 远程调用 |
|---|---|---|
| 方式 | JVM 内存方法调用 | 网络 HTTP 请求 |
| 性能 | 纳秒级 | 毫秒级（慢 1000 倍+） |
| 可能失败？ | 几乎不会 | 网络超时、服务挂了都会失败 |
| 需要序列化？ | 不需要 | 需要（对象 → JSON → 对象） |

## 三、RestTemplate 用法

Spring 提供的 HTTP 请求工具，使用分两步：

### 3.1 注册 Bean

```java
@Configuration
public class RemoteCallConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

### 3.2 发起远程调用

```java
@Autowired
private RestTemplate restTemplate;

// exchange — 万能方法，支持任意请求方式
ResponseEntity<List<ItemDTO>> response = restTemplate.exchange(
    "http://localhost:8081/items?ids={ids}",  // URL
    HttpMethod.GET,                            // 请求方式
    null,                                      // 请求体（GET 没有）
    new ParameterizedTypeReference<List<ItemDTO>>() {},  // 返回值类型
    Map.of("ids", CollUtil.join(itemIds, ","))  // URL 参数
);

// 检查状态码
if (!response.getStatusCode().is2xxSuccessful()) {
    return;
}
List<ItemDTO> items = response.getBody();
```

### 3.3 常用方法速查

| 方法 | 场景 |
|---|---|
| `getForObject` | 简单 GET，只要 body |
| `getForEntity` | GET，需要状态码/响应头 |
| `postForObject` | POST，只要 body |
| `exchange` | 万能，泛型类型、复杂请求都用它 |

## 四、ParameterizedTypeReference 原理

**问题**：Java 泛型擦除，运行时 `List<ItemDTO>` 变成 `List`，Jackson 不知道反序列化成什么。

**解决**：匿名类保留泛型信息。

```java
// 编译后生成匿名类，字节码里记录了 "extends ParameterizedTypeReference<List<ItemDTO>>"
new ParameterizedTypeReference<List<ItemDTO>>() {}

// Spring 通过反射读取：
Type type = getClass().getGenericSuperclass();  // 拿到完整泛型信息
// → 就能正确反序列化成 List<ItemDTO>
```

> 泛型擦除只擦**变量声明**，不擦**类定义**。匿名类的泛型信息保留在字节码里。

## 五、CartVO 默认值踩坑

拆分后购物车查询返回的 `newPrice`、`status`、`stock` 不是真实数据，而是 CartVO 的默认值：

```java
// CartVO.java
private Integer newPrice;       // 无默认 → null
private Integer status = 1;     // 默认 1
private Integer stock = 10;     // 默认 10
```

**原因**：`handleCartItems()` 原来通过 `itemService.queryItemByIds()` 填充这三个字段，拆分后这段代码被注释掉了，CartVO 永远带着默认值返回。

**解决**：用 RestTemplate 远程调用 item-service 获取商品最新信息，再填充到 CartVO。

## 六、实操：cart-service 远程调用

### 6.1 注册 RestTemplate

```java
@Configuration
public class RemoteCallConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

### 6.2 改造 handleCartItems

```java
@Autowired
private RestTemplate restTemplate;

private void handleCartItems(List<CartVO> vos) {
    // 1. 收集商品 ID
    Set<Long> itemIds = vos.stream().map(CartVO::getItemId).collect(Collectors.toSet());

    // 2. 远程调用 item-service
    ResponseEntity<List<ItemDTO>> response = restTemplate.exchange(
        "http://localhost:8081/items?ids={ids}",
        HttpMethod.GET, null,
        new ParameterizedTypeReference<List<ItemDTO>>() {},
        Map.of("ids", CollUtil.join(itemIds, ","))
    );

    if (!response.getStatusCode().is2xxSuccessful()) {
        return;
    }
    List<ItemDTO> items = response.getBody();
    if (CollUtils.isEmpty(items)) {
        return;
    }

    // 3. 填充 CartVO
    Map<Long, ItemDTO> itemMap = items.stream()
        .collect(Collectors.toMap(ItemDTO::getId, Function.identity()));
    for (CartVO v : vos) {
        ItemDTO item = itemMap.get(v.getItemId());
        if (item != null) {
            v.setNewPrice(item.getPrice());
            v.setStatus(item.getStatus());
            v.setStock(item.getStock());
        }
    }
}
```

### 6.3 踩坑

**Connection refused** — item-service 没启动，端口 8081 没人监听。先启动被调用方。

**URL 写错** — 确认 item-service 的 Controller 路径是 `/items`（不是 `/item`）。

## 七、RestTemplate vs Feign

| | RestTemplate | OpenFeign |
|---|---|---|
| 写法 | 手动拼 URL，手动调 exchange | 声明接口 + 注解，像调本地方法 |
| 可读性 | 一般 | 好 |
| 负载均衡 | 需要配合 `@LoadBalanced` + 服务名 | 内置支持 |
| 适合场景 | 简单调用、学习理解原理 | 生产项目推荐 |

> RestTemplate 是理解远程调用原理的好起点——本质就是发 HTTP 请求。后面学 Feign 会觉得更简单。

## 八、全文总结

1. **远程调用本质**：不同 JVM 之间通过 HTTP 请求通信
2. **RestTemplate**：Spring 提供的 HTTP 工具，`exchange` 是万能方法
3. **ParameterizedTypeReference**：利用匿名类保留泛型信息，解决反序列化问题
4. **CartVO 默认值**：拆分后 `handleCartItems()` 被注释，字段用默认值填充
5. **Connection refused**：被调用方没启动

## 九、关联笔记

- [[微服务拆分原则与实操]] — 拆分的理论和实操
- [[服务注册发现与Nacos]] — 解决 RestTemplate 硬编码地址的问题
