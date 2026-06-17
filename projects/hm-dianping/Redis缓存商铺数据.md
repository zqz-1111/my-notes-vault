---
date: 2026-06-17
tags: [hm-dianping, Redis, 缓存]
---

# 【Redis】缓存商铺数据｜Cache Aside 模式实战

## 一、写在前面

1. 为什么学这个 — 商铺查询是高频接口，每次都查数据库太慢，用 Redis 做缓存能大幅降低数据库压力
2. 学完这篇你能掌握 — Cache Aside 模式的读写实现、缓存更新策略选型、常见的缓存坑怎么避

## 二、前置准备

- Spring Boot 2.3.12 + Redis（localhost:6379）
- 已引入 `spring-boot-starter-data-redis`（自动装配 `StringRedisTemplate`）
- 了解 Redis 的 String 和 List 数据类型基本操作

## 三、正文分步讲解

### 3.1 读逻辑：先查缓存，再查数据库

核心思路：请求进来 → 先问 Redis 有没有 → 有就直接返回（缓存命中）→ 没有就查 MySQL，查到后写入 Redis 并设 TTL

```java
@Override
public Result queryById(Long id) {
    String key = CACHE_SHOP_KEY + id;
    // 1. 从 Redis 查缓存
    String shopJson = stringRedisTemplate.opsForValue().get(key);
    // 2. 缓存命中，直接返回
    if (StrUtil.isNotBlank(shopJson)) {
        Shop shop = JSONUtil.toBean(shopJson, Shop.class);
        return Result.ok(shop);
    }
    // 3. 缓存未命中，查数据库
    Shop shop = this.getById(id);
    if (Objects.isNull(shop)) {
        return Result.fail("店铺不存在");
    }
    // 4. 写入 Redis，设置 30 分钟过期
    stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop), CACHE_SHOP_TTL, TimeUnit.MINUTES);
    return Result.ok(shop);
}
```

### 3.2 写逻辑：先更新数据库，再删缓存

核心思路：更新时先改数据库，再删掉 Redis 缓存。下次读请求进来时会自动重建缓存。

```java
@Override
public Result updateByIdWithCache(Shop shop) {
    // 1. 先更新数据库
    updateById(shop);
    // 2. 再删除缓存
    String key = CACHE_SHOP_KEY + shop.getId();
    stringRedisTemplate.delete(key);
    return Result.ok();
}
```

> 为什么是「删缓存」而不是「更新缓存」？见下文方案对比。

## 四、方案对比

### 缓存更新策略对比

| 方案 | 做法 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|---|
| 懒加载（TTL 兜底） | 只在缓存未命中时写入，靠 TTL 过期 | 实现简单 | 过期前数据一直旧的 | 低一致性需求 |
| 更新数据库 + 更新缓存 | 改库后同步改缓存 | 实时一致 | 并发时容易写入脏数据 | 不推荐 |
| 更新数据库 + 删除缓存 ✅ | 改库后删缓存，下次读时重建 | 避免并发写入问题，实现简单 | 下次读多一次查库 | 高一致性需求 |

**选择：** 更新数据库 + 删除缓存方案，因为避免了并发更新缓存导致的数据不一致问题，且下次查询自动重建缓存，代价可控。

### 为什么「先更新数据库，后删缓存」？

```
❌ 先删缓存，再更新数据库：
  线程A：删缓存
  线程B：查缓存没了 → 查数据库（旧值）→ 写入缓存（旧值）
  线程A：更新数据库为新值
  结果：数据库新，缓存旧 ← 脏数据

✅ 先更新数据库，再删缓存：
  线程A：更新数据库为新值 → 删缓存
  线程B：查缓存没了 → 查数据库（新值）→ 写入缓存（新值）
  结果：一致
```

### @Transactional 需要吗？

**不需要。** `@Transactional` 只管数据库事务，Redis 不在 Spring 事务管理范围内。这里只有一条数据库 SQL + 一个 Redis 操作，写了也没用。只有多条数据库操作需要原子性时才需要。

## 五、踩坑避坑指南

### 逻辑坑（不报错但会出问题）

**坑 1：缓存没设 TTL**
- 问题描述：`set(key, json)` 没加过期时间，缓存永久生效
- 后果：数据库改了数据，Redis 里永远是旧值，造成缓存与数据库不一致
- 正确做法：`set(key, json, CACHE_SHOP_TTL, TimeUnit.MINUTES)` 加上过期时间兜底

**坑 2：缓存穿透 — 数据库查不到的数据没缓存**
- 问题描述：用不存在的 id 查询，每次都打到数据库
- 后果：恶意请求可以直接打崩数据库
- 正确做法：数据库查不到时，缓存空值（空字符串）+ 短 TTL（如 2 分钟）

**坑 3：Redis 常量忘了定义**
- 问题描述：代码里用了 `CACHE_SHOP_TYPE_KEY`，但 `RedisConstants.java` 里没有这个常量
- 后果：编译报错，IDE 爆红
- 正确做法：用到新的 Redis key 前，先在 `RedisConstants.java` 里定义好前缀和 TTL

### 报错坑

**坑 4：TTL 刷新时 key 拼写错误**
- 报错信息：不报错，但 TTL 刷新无效，登录态提前过期
- 原因分析：`RefreshTokenInterceptor` 中用 `stringRedisTemplate.expire(token, ...)` 传了原始 token 字符串，而不是拼接了前缀的 `tokenKey`
- 解决方案：改成 `stringRedisTemplate.expire(tokenKey, ...)`

## 六、全文总结

- **Cache Aside 模式**：读时先查缓存，未命中则查库写缓存；写时先更新库，再删缓存
- **TTL 兜底**：所有缓存都要设过期时间，防止永久脏数据
- **先库后缓存**：更新时先改库再删缓存，避免并发写入的不一致问题
- **不需要 @Transactional**：Redis 操作不在数据库事务范围内
- 下一步可以学习：缓存穿透的空值缓存、缓存雪崩、布隆过滤器

## 七、关联笔记

- [[登录功能完整实现]] — 登录拦截器重构也在本次学习中完成
- [[Redis基础入门]] — Redis 基础数据结构和命令
- [[Redis的Java客户端]] — Jedis 和 SpringDataRedis

## 八、代码变更

- 修改：`src/main/java/com/hmdp/controller/ShopController.java` — 调用新的缓存查询和更新方法
- 修改：`src/main/java/com/hmdp/service/IShopService.java` — 新增 `queryById`、`updateByIdWithCache` 方法声明
- 修改：`src/main/java/com/hmdp/service/impl/ShopServiceImpl.java` — 实现缓存读写逻辑
- 修改：`src/main/java/com/hmdp/utils/RedisConstants.java` — 新增 `CACHE_SHOP_TYPE_KEY` 常量
- 修改：`src/main/java/com/hmdp/service/impl/ShopTypeServiceImpl.java` — 补充 TTL 和静态导入
- 修改：`src/main/java/com/hmdp/utils/RefreshTokenInterceptor.java` — 修复 TTL key bug
