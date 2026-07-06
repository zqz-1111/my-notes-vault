---
date: 2026-07-06
tags: [SpringCloud, Gateway, OpenFeign, 微服务, 黑马商城]
---

# 【Gateway】微服务获取用户与OpenFeign传递

## 一、写在前面

1. 之前网关已经能校验 Token 并解析 userId，但 userId 只打印了没传给下游——这篇解决"用户信息怎么跨服务传递"
2. 学完这篇你能掌握：网关怎么把用户信息传给微服务、微服务怎么用拦截器接住、Feign 调用怎么自动携带用户信息

## 二、问题背景

### 当时的困境

```
用户请求 → Gateway（解析出 userId = 123）
              ↓
         trade-service（不知道用户是谁！）
              ↓
         cart-service（更不知道了！）
```

**核心矛盾：** Token 只在网关解析了一次，但下游多个微服务都需要知道"当前用户是谁"。

### 解决思路

**请求头传递 + ThreadLocal 存储 + Feign 拦截器转发**

```
Gateway：解析 Token → userId → 塞到请求头 user-info
              ↓
微服务A：拦截器读 user-info → 存入 UserContext (ThreadLocal)
              ↓
微服务A 用 Feign 调用微服务B：Feign 拦截器从 UserContext 取 → 塞到 Feign 请求头
              ↓
微服务B：同样读 user-info → 存入 UserContext
```

## 三、三个核心组件

### 组件一：Gateway AuthGlobalFilter — 写入请求头

```java
// 5.传递用户信息到下游微服务
ServerHttpRequest newRequest = request.mutate()
        .header("user-info", userId.toString())
        .build();
ServerWebExchange newExchange = exchange.mutate()
        .request(newRequest)
        .build();
// 6.放行
return chain.filter(newExchange);
```

**关键点：**
- `request.mutate()` 创建新请求，不修改原请求（不可变设计）
- 请求头名称用 `user-info`，全链路保持一致

### 组件二：UserInfoInterceptor — 从请求头读取

**位置：** `hm-common` 模块（所有微服务都能用）

```java
public class UserInfoInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 1.获取请求头中的用户信息
        String userInfo = request.getHeader("user-info");
        // 2.判断是否为空
        if (StrUtil.isNotBlank(userInfo)) {
            // 不为空，保存到ThreadLocal
            UserContext.setUser(Long.valueOf(userInfo));
        }
        // 3.放行
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        // 移除用户
        UserContext.removeUser();
    }
}
```

**设计亮点：**
- 放在 `hm-common`，所有微服务复用
- 不重复解析 Token，只从请求头读取
- `StrUtil.isNotBlank` 防空指针
- `afterCompletion` 清理 ThreadLocal 防内存泄漏

### 组件三：Feign RequestInterceptor — 自动携带用户信息

**位置：** `hm-api` 模块的 `DefaultFeignConfig`

```java
@Bean
public RequestInterceptor userInfoRequestInterceptor(){
    return new RequestInterceptor() {
        @Override
        public void apply(RequestTemplate template) {
            // 获取登录用户
            Long userId = UserContext.getUser();
            if(userId == null) {
                // 如果为空则直接跳过
                return;
            }
            // 如果不为空则放入请求头中，传递给下游微服务
            template.header("user-info", userId.toString());
        }
    };
}
```

**作用：** 每次 Feign 调用自动执行，从 ThreadLocal 取 userId → 塞到 Feign 请求头

## 四、完整链路图

```
1. 用户请求 (带 Token: Bearer xxx)
   ↓
2. Gateway AuthGlobalFilter
   ├─ 校验 Token ✅
   ├─ 解析 userId = 123
   ├─ 写入请求头: user-info = 123
   ↓
3. 转发到微服务A (如 trade-service)
   ↓
4. UserInfoInterceptor (来自 hm-common)
   ├─ 读请求头: user-info = 123
   ├─ UserContext.setUser(123L)
   ↓
5. Controller/Service 业务代码
   ├─ UserContext.getUser() = 123
   ├─ 需要调用其他微服务
   ↓
6. FeignClient 调用 (如 cartClient.removeByItemIds())
   ↓
7. Feign RequestInterceptor
   ├─ UserContext.getUser() = 123
   ├─ 写入 Feign 请求头: user-info = 123
   ↓
8. 转发到微服务B (如 cart-service)
   ↓
9. UserInfoInterceptor (同样来自 hm-common)
   ├─ 读请求头: user-info = 123
   ├─ UserContext.setUser(123L)
   ↓
10. 业务代码正常执行
```

## 五、实际场景：下单流程

```
用户下单 → trade-service
  ├─ UserContext.getUser() = 123
  ├─ 创建订单
  ├─ itemClient.updateStock(...)  // Feign 调用商品服务
  │   └─ 自动携带 user-info: 123
  ├─ cartClient.removeByItemIds(...)  // Feign 调用购物车服务
  │   └─ 自动携带 user-info: 123
  └─ 返回订单ID
```

## 六、恢复购物车代码

之前因为无法获取登录用户，写死了用户ID：

```java
// 改前：写死用户ID
List<Cart> carts = lambdaQuery().eq(Cart::getUserId, 1L /*TODO UserContext.getUser()*/).list();

// 改后：从UserContext获取
List<Cart> carts = lambdaQuery().eq(Cart::getUserId, UserContext.getUser()).list();
```

## 七、踩坑避坑指南

### 报错坑：hm-api 找不到 UserContext

- 报错信息：`无法解析符号 'UserContext'`
- 原因：`hm-api` 模块没有依赖 `hm-common`
- 解决方案：在 `hm-api` 的 `pom.xml` 添加依赖

```xml
<dependency>
    <groupId>com.heima</groupId>
    <artifactId>hm-common</artifactId>
    <version>1.0.0</version>
</dependency>
```

## 八、效果验证

| 测试场景 | 预期结果 |
|---------|---------|
| Gateway 转发请求到微服务 | 微服务能从 `user-info` 请求头拿到 userId |
| trade-service 调用 cart-service | cart-service 能拿到 userId，正确清空购物车 |
| 查询购物车 | 返回当前用户的购物车，而不是用户1的 |

## 九、全文总结

- **核心思想：** Token 只在网关解析一次，之后全链路用请求头传 userId
- **三个组件：** Gateway 写入 → 拦截器读取 → Feign 转发
- **存储介质：** 请求头 `user-info` + ThreadLocal `UserContext`
- **复用设计：** `UserInfoInterceptor` 放 `hm-common`，`RequestInterceptor` 放 `hm-api`

## 十、关联笔记

- [[Gateway鉴权过滤器实现]]
- [[Gateway入门与路由配置]]
- [[OpenFeign快速入门]]

## 十一、代码变更

- 修改：`hm-gateway/src/main/java/com/hmall/gateway/filter/AuthGlobalFilter.java` — 写入 user-info 请求头
- 新增：`hm-common/src/main/java/com/hmall/common/interceptor/UserInfoInterceptor.java` — 从请求头读取用户信息
- 修改：`hm-api/src/main/java/com/hmall/api/config/DefaultFeignConfig.java` — 添加 Feign 拦截器
- 修改：`hm-api/pom.xml` — 添加 hm-common 依赖
- 修改：`cart-service/src/main/java/com/hmall/cart/service/impl/CartServiceImpl.java` — 恢复 UserContext.getUser()
