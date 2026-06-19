---
date: '2026-06-19'
tags:
  - hm-dianping
  - 缓存
  - Bug修复
  - 缓存击穿
---

# 【Bug修复】CacheClient 缓存击穿空值问题

## 一、写在前面

1. 为什么学这个 — 切换缓存策略时踩了个坑，理解不同缓存方法的数据格式差异很重要
2. 学完这篇你能掌握什么 — 缓存方法之间的数据格式兼容问题，以及如何做防御性编程

## 二、问题现象

调用 `/shop/1` 查询店铺详情，后端报错：

```
java.lang.NullPointerException: null
    at com.hmdp.utils.CacheClient.handleCacheBreakdown(CacheClient.java:138)
```

## 三、排查过程

### 3.1 定位报错行

```java
RedisData redisData = JSONUtil.toBean(jsonStr, RedisData.class);
JSONObject data = (JSONObject) redisData.getData();
T t = JSONUtil.toBean(data, type);
LocalDateTime expireTime = redisData.getExpireTime();
if (expireTime.isAfter(LocalDateTime.now())) {  // ← 第138行，expireTime 是 null
```

### 3.2 根因分析

两个方法存的数据格式不一样：

| 方法 | 存储格式 | 示例 |
|---|---|---|
| `handleCachePenetration` | 裸对象 JSON | `{"id":1,"name":"xxx"}` |
| `handleCacheBreakdown` | RedisData 包装 | `{"data":{...},"expireTime":"2026-..."}` |

之前用 `handleCachePenetration` 缓存过这个 shop，Redis 里存的是裸 JSON。现在换成 `handleCacheBreakdown` 去读，反序列化成 `RedisData` 后 `expireTime` 字段自然是 null。

### 3.3 另一个问题

清掉旧缓存后，`handleCacheBreakdown` 直接返回 null：

```java
if (StrUtil.isBlank(jsonStr)) {
    return null;  // 缓存为空直接返回，不查数据库！
}
```

这个方法只处理了"缓存有数据但过期"的场景，缓存为空时不会回源查数据库。

## 四、解决方案

修改 `handleCacheBreakdown`，缓存未命中时回源查数据库并写入正确格式的缓存：

```java
if (StrUtil.isBlank(jsonStr)) {
    // 缓存未命中，回源查数据库，写入逻辑过期格式的缓存
    T t = dbFallback.apply(id);
    if (Objects.isNull(t)) {
        return null;
    }
    this.setWithLogicalExpire(key, t, timeout, unit);
    return t;
}
```

## 五、踩坑避坑指南

### 逻辑坑：缓存方法之间的格式不兼容

- 问题：不同缓存方法存储的数据格式不同，混用会出问题
- 后果：反序列化失败或字段为 null
- 正确做法：切换缓存策略时，先清掉旧格式的缓存数据

### 逻辑坑：缓存击穿方法不处理缓存为空

- 问题：`handleCacheBreakdown` 只处理"有数据但过期"，不处理"没数据"
- 后果：手动清缓存或首次访问时直接返回 null
- 正确做法：缓存为空时回源查数据库，写入正确格式的缓存

## 六、效果验证

1. 重启项目
2. 访问 `/shop/1`
3. 第一次请求：查数据库 → 写入 RedisData 格式缓存 → 返回数据
4. 后续请求：命中缓存，判断是否过期

## 七、全文总结

- `handleCachePenetration` 存裸 JSON，`handleCacheBreakdown` 存 RedisData 包装，格式不兼容
- 切换缓存策略时要清掉旧缓存
- 缓存方法要考虑"缓存为空"的场景，不能只处理"有数据但过期"

## 八、关联笔记

- [[Redis缓存三大经典问题]] — 缓存穿透、击穿、雪崩的理论基础
- [[Redis缓存商铺数据]] — Cache Aside 模式

## 九、代码变更

- 修改：`src/main/java/com/hmdp/utils/CacheClient.java` — handleCacheBreakdown 方法增加缓存为空时的回源查数据库逻辑
