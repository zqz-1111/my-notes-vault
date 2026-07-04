---
date: 2026-07-04
tags: [黑马商城微服务, SpringCloud, OpenFeign, 远程调用]
---

# OpenFeign 快速入门

## 一、写在前面

上一篇用 RestTemplate + DiscoveryClient 实现了服务发现和远程调用，但代码很繁琐：

```java
// 1. 获取实例列表
List<ServiceInstance> instances = discoveryClient.getInstances("item-service");
// 2. 负载均衡选一个
ServiceInstance instance = instances.get(ThreadLocalRandom.current().nextInt(instances.size()));
// 3. 拼 URL
String url = instance.getUri() + "/items?ids={ids}";
// 4. 发请求
ResponseEntity<List<ItemDTO>> response = restTemplate.exchange(url, HttpMethod.GET, null, ...);
// 5. 解析响应
List<ItemDTO> items = response.getBody();
```

每次远程调用都要写这一堆，而且跟本地方法调用风格差异很大。

**OpenFeign** 就是来解决这个问题的——让远程调用像调本地方法一样简单。

## 二、OpenFeign 是什么

**一句话**：声明式 HTTP 客户端。

你只需要写一个**接口**，用注解声明「调谁、怎么调、参数是啥、返回啥」，框架自动生成实现类帮你发请求。

### 2.1 对比

| 方式 | 代码量 | 编程体验 |
|---|---|---|
| RestTemplate | 多（手动拼 URL、发请求、解析） | 一会儿远程一会儿本地，割裂 |
| OpenFeign | 少（声明接口 + 注解） | 像调本地方法，统一 |

### 2.2 核心原理

```
你写的接口                     框架生成的实现类（动态代理）
┌─────────────────┐           ┌─────────────────────────────────┐
│ @FeignClient    │           │ class ItemClient$Proxy {        │
│ public interface│    ───→   │   List<ItemDTO> queryByIds() { │
│   ItemClient {  │  动态代理  │     // 1. 拼 URL               │
│     @GetMapping │           │     // 2. 发 HTTP 请求          │
│     ...         │           │     // 3. 解析响应              │
│   }             │           │   }                             │
└─────────────────┘           └─────────────────────────────────┘
```

**关键点**：OpenFeign 用 SpringMVC 注解（`@GetMapping`、`@PostMapping`、`@RequestParam` 等）声明调用信息，框架根据注解自动生成远程调用代码。

## 三、使用步骤

### 3.1 引入依赖

```xml
<!--openFeign-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<!--负载均衡（OpenFeign 依赖 LoadBalancer）-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

### 3.2 启动类开启 Feign

```java
@EnableFeignClients  // 开启 Feign 功能
@SpringBootApplication
public class CartApplication {
    public static void main(String[] args) {
        SpringApplication.run(CartApplication.class, args);
    }
}
```

### 3.3 编写 Feign 客户端接口

```java
@FeignClient("item-service")  // 声明调用哪个服务
public interface ItemClient {

    @GetMapping("/items")      // 请求方式 + 路径
    List<ItemDTO> queryItemByIds(@RequestParam("ids") Collection<Long> ids);
}
```

**注解说明：**

| 注解 | 作用 | 对应 RestTemplate 写法 |
|---|---|---|
| `@FeignClient("item-service")` | 指定调用的服务名 | `discoveryClient.getInstances("item-service")` |
| `@GetMapping("/items")` | 请求方式 + 路径 | `HttpMethod.GET` + `"/items"` |
| `@RequestParam("ids")` | 请求参数 | `Map.of("ids", ...)` |
| 返回值 `List<ItemDTO>` | 响应体类型 | `new ParameterizedTypeReference<List<ItemDTO>>(){}` |

### 3.4 注入使用

```java
@Service
@RequiredArgsConstructor
public class CartServiceImpl extends ServiceImpl<CartMapper, Cart> implements ICartService {

    private final ItemClient itemClient;  // 注入 Feign 客户端

    private void handleCartItems(List<CartVO> vos) {
        Set<Long> itemIds = vos.stream().map(CartVO::getItemId).collect(Collectors.toSet());

        // 一行搞定！就像调本地方法
        List<ItemDTO> items = itemClient.queryItemByIds(itemIds);

        // ... 后续处理
    }
}
```

## 四、对比总结

```java
// ========== 之前：RestTemplate（10行） ==========
List<ServiceInstance> instances = discoveryClient.getInstances("item-service");
ServiceInstance instance = instances.get(ThreadLocalRandom.current().nextInt(instances.size()));
ResponseEntity<List<ItemDTO>> response = restTemplate.exchange(
    instance.getUri() + "/items?ids={ids}",
    HttpMethod.GET, null,
    new ParameterizedTypeReference<List<ItemDTO>>() {},
    Map.of("ids", CollUtil.join(itemIds, ","))
);
List<ItemDTO> items = response.getBody();

// ========== 之后：OpenFeign（1行） ==========
List<ItemDTO> items = itemClient.queryItemByIds(itemIds);
```

## 五、踩坑记录

**坑 1：忘记 @EnableFeignClients**
- 报错：`Field itemClient required a bean of type 'ItemClient' that could not be found`
- 解决：启动类加 `@EnableFeignClients`

**坑 2：忘记 loadbalancer 依赖**
- 报错：`No instances available for item-service`
- 原因：OpenFeign 需要 loadbalancer 做负载均衡
- 解决：pom.xml 加 `spring-cloud-starter-loadbalancer`

## 六、进阶：OKHttp 连接池

### 6.1 为什么要用连接池？

| 客户端 | 连接池 | 说明 |
|---|---|---|
| HttpURLConnection | ❌ 默认 | 每次请求都新建连接、关闭连接 |
| Apache HttpClient | ✅ | 复用连接，性能好 |
| OKHttp | ✅ | 复用连接，性能好，SpringCloud 推荐 |

**类比**：
- 无连接池 = 每次打车都重新叫车
- 有连接池 = 包个司机，随时出发

### 6.2 配置步骤

**pom.xml 添加依赖：**
```xml
<!--OK http 的依赖 -->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
</dependency>
```

**application.yaml 开启：**
```yaml
feign:
  okhttp:
    enabled: true # 开启OKHttp功能
```

重启即可生效。

## 七、进阶：最佳实践（抽取公共模块）

### 7.1 问题

多个服务都需要调用 item-service，每个服务都写一份 ItemClient？重复代码。

### 7.2 解决方案

抽取 Feign 客户端到独立模块 `hm-api`，所有服务共用：

```
项目结构：
hmall/
├── hm-common      — 公共工具、异常处理
├── hm-api         — Feign 客户端、通用 DTO（新建）
├── item-service
├── cart-service
└── trade-service（将来）
```

### 7.3 实操步骤

**第一步：创建 hm-api 模块**

pom.xml 依赖：
```xml
<dependencies>
    <!--open feign-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <!-- load balancer-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
</dependencies>
```

**第二步：把 ItemClient 和 ItemDTO 移到 hm-api**

```
hm-api/src/main/java/com/hmall/api/
├── client/
│   └── ItemClient.java
├── dto/
│   └── ItemDTO.java
└── config/
    └── DefaultFeignConfig.java
```

**第三步：cart-service 依赖 hm-api**

```xml
<!--feign模块-->
<dependency>
    <groupId>com.heima</groupId>
    <artifactId>hm-api</artifactId>
    <version>1.0.0</version>
</dependency>
```

**第四步：更新 import + 扫描包**

```java
// import 改成 hm-api 的包
import com.hmall.api.client.ItemClient;
import com.hmall.api.dto.ItemDTO;

// 启动类指定扫描 hm-api 的包
@EnableFeignClients(value = "com.hmall.api", defaultConfiguration = DefaultFeignConfig.class)
```

### 7.4 好处

- ItemClient 只写一次，cart-service、trade-service 共用
- DTO 也共用，不用重复定义
- 改一处全生效

## 八、进阶：Feign 日志配置

### 8.1 日志级别

| 级别 | 内容 | 场景 |
|---|---|---|
| NONE | 不记录 | 生产环境（默认） |
| BASIC | 请求方法、URL、状态码、耗时 | 日常开发 |
| HEADERS | BASIC + 请求响应头 | 进阶调试 |
| FULL | 所有细节 | 排查 bug |

### 8.2 配置方式

**第一步：创建配置类**

```java
public class DefaultFeignConfig {
    @Bean
    public Logger.Level feignLogLevel(){
        return Logger.Level.FULL;  // 开发用 FULL，生产改成 NONE
    }
}
```

**第二步：应用配置**

```java
// 全局生效（所有 Feign 客户端）
@EnableFeignClients(value = "com.hmall.api", defaultConfiguration = DefaultFeignConfig.class)

// 或局部生效（单个客户端）
@FeignClient(value = "item-service", configuration = DefaultFeignConfig.class)
```

**第三步：配置日志级别**

Feign 日志需要 `DEBUG` 级别才输出，application.yaml：
```yaml
logging:
  level:
    com.hmall: debug
```

### 8.3 输出示例

```
---> GET http://item-service/items?ids=100000006163 HTTP/1.1
---> END HTTP (0-byte body)

<--- HTTP/1.1 200 OK (58ms)
<--- END HTTP (308-byte body)
```

## 九、全文总结

1. **OpenFeign 解决什么**：简化远程调用，像调本地方法一样调远程接口
2. **核心原理**：接口 + SpringMVC 注解声明调用信息，框架动态代理生成实现类
3. **使用步骤**：加依赖 → `@EnableFeignClients` → 写接口（`@FeignClient` + `@GetMapping`） → 注入使用
4. **连接池**：用 OKHttp 替代默认 HttpURLConnection，复用连接提升性能
5. **最佳实践**：抽取 Feign 客户端到 hm-api 公共模块，避免重复代码
6. **日志配置**：开发环境开 FULL，生产环境关掉

## 十、关联笔记

- [[服务注册发现与Nacos]] — 前置知识，Nacos 配置和服务发现
- [[远程调用与RestTemplate]] — OpenFeign 替代方案

## 八、代码变更

- 修改：`cart-service/pom.xml` — 添加 openfeign、loadbalancer 依赖
- 新增：`cart-service/.../ItemClient.java` — Feign 客户端接口
- 修改：`cart-service/.../CartServiceImpl.java` — 注入 ItemClient，替换 RestTemplate 调用
