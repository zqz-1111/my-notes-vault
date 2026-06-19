---
date: 2026-06-18
tags: [hm-dianping, Redis, 缓存]
---

# Redis 缓存三大经典问题：穿透、雪崩、击穿

## 一、写在前面

1. 为什么学这个：高并发场景下，缓存是数据库的"挡箭牌"，但这面盾牌有三种经典的破法——穿透、雪崩、击穿。不理解它们，缓存用得再溜也可能翻车。
2. 学完这篇你能掌握：三种缓存问题的定义、区别、触发场景，以及行业通用的解决方案，能应对面试和实际开发。

## 二、前置准备

- 已了解 Redis 基本操作（String 类型的 get/set）
- 已了解缓存的基本概念（缓存命中、未命中、TTL）
- 项目环境：Java 8 + Spring Boot + Redis

## 三、正文讲解

### 3.1 缓存穿透 —— 查的东西根本不存在

#### 定义

用户请求的数据在缓存和数据库中**都不存在**，缓存永远"挡不住"，请求直接穿透到数据库。

#### 类比

你去图书馆找一本根本不存在的书，管理员每次都说"没有"，但你每隔 5 分钟就来问一次，管理员每次都得翻一遍库存表。

#### 触发场景

- 恶意攻击：用 `-1`、`9999999` 这种不存在的 id 请求
- 业务 bug：参数校验没做好

#### 解决方案

| 方案 | 思路 | 优点 | 缺点 |
|---|---|---|---|
| **缓存空对象** | 数据库没有，就在 Redis 存个空值，设短 TTL | 实现简单 | 额外内存占用、短期数据不一致 |
| **布隆过滤器** | 在缓存前加一层"白名单"，不存在的 key 直接拦截 | 内存占用极低 | 实现复杂、存在误判 |
| **入口防御** | 增强 id 复杂度、参数校验、权限校验、限流 | 从源头减少恶意请求 | 需要额外开发 |

#### 缓存空对象代码实现

```java
@Override
public Result queryById(Long id) {
    String key = CACHE_SHOP_KEY + id;
    // 1. 从 Redis 查询缓存
    String shopJson = stringRedisTemplate.opsForValue().get(key);

    // 2. 缓存命中有效数据，直接返回
    if (StrUtil.isNotBlank(shopJson)) {
        Shop shop = JSONUtil.toBean(shopJson, Shop.class);
        return Result.ok(shop);
    }

    // 3. 缓存命中空对象（空字符串），返回不存在
    if (shopJson != null) {
        return Result.fail("店铺不存在");
    }

    // 4. 缓存未命中，查数据库
    Shop shop = this.getById(id);

    // 5. 数据库也没有，缓存空对象，TTL 2秒
    if (shop == null) {
        stringRedisTemplate.opsForValue().set(key, "", CACHE_NULL_TTL, TimeUnit.SECONDS);
        return Result.fail("店铺不存在");
    }

    // 6. 数据库有，写入缓存
    stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop),
            CACHE_SHOP_TTL, TimeUnit.MINUTES);
    return Result.ok(shop);
}
```

**关键点：**
- 空对象的 TTL 设为 2 秒（`CACHE_NULL_TTL`），避免长期数据不一致
- 判断顺序：有效值 → 空对象 → 查数据库

---

### 3.2 缓存雪崩 —— 大面积塌方

#### 定义

同一时段内大量缓存 key 集中失效，或 Redis 服务整体宕机，导致海量请求直接穿透到数据库。

#### 类比

超市所有自助结账机同时坏了，几千个顾客全涌到人工窗口，收银员被挤爆。

#### 两种触发场景

**场景一：大批 key 同时过期**

```
00:00  批量导入 10000 个商品，TTL 统一设 1 小时
01:00  10000 个 key 同时失效 → 瞬间打爆数据库
```

**场景二：Redis 整体宕机**

```
Redis 挂了 → 所有请求绕过缓存 → 全部打到数据库 → 数据库崩了
```

#### 解决方案

| 方案 | 解决什么问题 | 实现方式 |
|---|---|---|
| **TTL 加随机值** | 防止集中过期 | 基础时间 + 随机偏移量 |
| **Redis 集群** | 防止单点故障 | 哨兵模式、Cluster 集群 |
| **降级限流** | 兜底保护数据库 | 网关限流、Sentinel |
| **多级缓存** | 层层拦截 | 本地缓存（Caffeine）+ Redis |

#### TTL 加随机值代码示例

```java
// 错误写法：统一 1 小时过期
stringRedisTemplate.expire(key, 60, TimeUnit.MINUTES);

// 正确写法：基础 1 小时 + 随机 0~10 分钟
int randomMinutes = new Random().nextInt(10);
stringRedisTemplate.expire(key, 60 + randomMinutes, TimeUnit.MINUTES);
```

---

### 3.3 缓存击穿 —— 热点 key 被打穿

#### 定义

单个热点 key 缓存失效的瞬间，海量并发请求同时打到数据库，导致数据库瞬时压力剧增。

#### 类比

限量球鞋店门口的"已售罄"牌子被风吹走了，1000 个人同时冲进店里问有没有鞋。

**核心特点：** 数据在数据库里**确实存在**，只是缓存临时失效。

#### 三种解决方案

**方案一：互斥锁（让请求排队）**

```
请求 1 → 缓存未命中 → 加锁成功 → 查数据库 → 写缓存 → 释放锁
请求 2 → 缓存未命中 → 加锁失败 → 等待...
请求 3 → 缓存未命中 → 加锁失败 → 等待...
                   ↓
请求 1 完成后，请求 2/3 → 缓存命中 → 直接返回
```

**代码实现（已完成）：**

```java
// 入口方法
@Override
public Result queryById(Long id) {
    // 互斥锁解决缓存击穿
    Shop shop = queryWithMutex(id);
    if (shop == null) {
        return Result.fail("店铺不存在！");
    }
    return Result.ok(shop);
}

// 互斥锁方法
public Shop queryWithMutex(Long id) {
    String key = RedisConstants.CACHE_SHOP_KEY + id;
    // 1. 从 Redis 查询缓存
    String shopJson = stringRedisTemplate.opsForValue().get(key);

    // 2. 命中有效数据，直接返回
    if (StrUtil.isNotBlank(shopJson)) {
        Shop shop = JSONUtil.toBean(shopJson, Shop.class);
        return shop;
    }

    // 3. 命中空值（防穿透），返回 null
    if (shopJson != null) {
        return null;
    }

    // 4. 缓存未命中，实现缓存重建
    String lockKey = RedisConstants.LOCK_SHOP_KEY + id;
    Shop shop = null;
    try {
        // 4.1 获取互斥锁
        boolean islock = tryLock(lockKey);

        // 4.2 获取失败，休眠并重试
        if (!islock) {
            Thread.sleep(50);
            return queryWithMutex(id);  // 递归重试
        }

        // 4.3 获取成功，查数据库
        shop = getById(id);

        if (shop == null) {
            // 数据库也没有，缓存空对象
            stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop),
                    RedisConstants.CACHE_NULL_TTL, TimeUnit.MINUTES);
            return null;
        }

        // 4.4 数据库有，写入缓存
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop),
                RedisConstants.CACHE_SHOP_TTL, TimeUnit.MINUTES);

    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    } finally {
        // 5. 释放互斥锁
        unlock(lockKey);
    }

    return shop;
}

// 加锁方法（SETNX）
private boolean tryLock(String key) {
    Boolean flag = stringRedisTemplate.opsForValue()
        .setIfAbsent(key, "1", 10, TimeUnit.SECONDS);
    return BooleanUtil.isTrue(flag);
}

// 解锁方法
private void unlock(String key) {
    stringRedisTemplate.delete(key);
}
```

**核心要点：**
- `setIfAbsent` = Redis 的 `SETNX`，key 不存在才设置成功，实现互斥
- 锁的 TTL 设 10 秒，防止死锁
- 没拿到锁 → sleep 50ms → 递归重试
- `finally` 确保锁一定释放

**方案二：逻辑过期 + 异步更新（不阻塞请求）**

```
请求 1 → 读缓存 → 发现逻辑过期 → 返回旧数据 + 异步更新缓存
请求 2 → 读缓存 → 发现逻辑过期 → 返回旧数据（或新数据）
```

**核心思想：** 缓存不设真正的 TTL，在 value 里存一个"逻辑过期时间"。读到数据发现过期了，就返回旧数据，同时异步去更新缓存。

**代码实现（已完成，注释备用）：**

```java
// 线程池用于异步重建缓存
private static final ExecutorService CACHE_REBUILD_EXECUTOR = 
    Executors.newFixedThreadPool(10);

public Shop queryWithLogicalExpire(Long id) {
    String key = CACHE_SHOP_KEY + id;
    // 1. 从 Redis 查询缓存
    String json = stringRedisTemplate.opsForValue().get(key);
    
    // 2. 不存在，直接返回
    if (StrUtil.isBlank(json)) {
        return null;
    }
    
    // 3. 命中，反序列化
    RedisData redisData = JSONUtil.toBean(json, RedisData.class);
    Shop shop = JSONUtil.toBean((JSONObject) redisData.getData(), Shop.class);
    LocalDateTime expireTime = redisData.getExpireTime();
    
    // 4. 判断是否过期
    if (expireTime.isAfter(LocalDateTime.now())) {
        // 4.1 未过期，直接返回
        return shop;
    }
    
    // 4.2 已过期，需要缓存重建
    String lockKey = LOCK_SHOP_KEY + id;
    boolean isLock = tryLock(lockKey);
    
    // 5. 获取到锁才开启异步重建
    if (isLock) {
        CACHE_REBUILD_EXECUTOR.submit(() -> {
            try {
                this.saveShop2Redis(id, 20L);  // 重建缓存
            } catch (Exception e) {
                throw new RuntimeException(e);
            } finally {
                unlock(lockKey);
            }
        });
    }
    
    // 6. 不管有没有获取到锁，都返回旧数据
    return shop;
}

// 预热缓存时调用：把数据写入 Redis 并设置逻辑过期时间
public void saveShop2Redis(Long id, Long expiredSeconds) {
    Shop shop = getById(id);
    RedisData redisData = new RedisData();
    redisData.setData(shop);
    redisData.setExpireTime(LocalDateTime.now().plusSeconds(expiredSeconds));
    // 注意：Redis key 本身永不过期，靠逻辑时间判断
    stringRedisTemplate.opsForValue().set(
        CACHE_SHOP_KEY + id, JSONUtil.toJsonStr(redisData));
}
```

**核心要点：**
- Redis key **永不过期**，在 value 里存逻辑过期时间
- 过期后返回旧数据，异步重建缓存（用线程池）
- 不阻塞用户请求，高并发场景性能更好

**方案三：热点数据永不过期（从根源避免）**

```
热点数据 → 不设 TTL → 后台定时任务主动更新
```

**适用场景：** 秒杀商品、热搜等核心热点数据。

#### 三种解法对比

| 解法 | 是否阻塞 | 数据一致性 | 适用场景 |
|---|---|---|---|
| **互斥锁** | 是（等待） | 强一致 | 对一致性要求高、并发量中等 |
| **逻辑过期** | 否（返回旧数据） | 短暂不一致 | 高并发、允许短暂数据延迟 |
| **永不过期** | 否 | 取决于更新频率 | 核心热点数据 |

---

## 四、三大问题对比总结

| 问题 | 核心诱因 | 数据是否存在 | 影响范围 |
|---|---|---|---|
| **缓存穿透** | 查询不存在的数据 | 缓存、数据库均不存在 | 无效请求持续打库 |
| **缓存击穿** | 热点 key 突然过期 | 数据库存在，缓存临时失效 | 单个热点数据 |
| **缓存雪崩** | 大批 key 同时过期 / Redis 宕机 | 数据均存在 | 全局大面积打库 |

**记忆口诀：**
- 穿透 = 东西不存在，穿过去了
- 击穿 = 盾牌被打穿一个点
- 雪崩 = 整个盾牌塌了

---

## 五、踩坑避坑指南

### 逻辑坑

**坑 1：缓存空对象 TTL 设太长**
- 问题：TTL 设成几小时甚至几天，数据库新增了数据但缓存还是空值
- 正确做法：TTL 设短一点，2~5 秒足够

**坑 2：批量设置缓存不加随机值**
- 问题：所有 key 同时过期，引发雪崩
- 正确做法：TTL 基础值 + 随机偏移量

**坑 3：互斥锁忘记释放**
- 问题：第一个请求异常退出，锁没释放，其他请求永远等待
- 正确做法：用 try-finally 确保释放锁

**坑 4：方法名大小写写错**
- 报错信息：`无法解析 'ShopServiceImpl' 中的方法 'trylock'`
- 原因：Java 大小写敏感，调用 `trylock` 但定义是 `tryLock`（大写 L）
- 解决方案：保持调用和定义的大小写一致，IDE 通常会提示

---

## 六、效果验证

- 用不存在的 id 请求接口，观察 Redis 是否写入了空值（`GET cache:shop:999999`）
- 等 2 秒后再请求，观察空值是否过期
- 用 JMeter 模拟高并发，观察数据库压力

---

## 七、全文总结

1. **缓存穿透**：查的东西不存在，用缓存空对象或布隆过滤器解决
2. **缓存雪崩**：大面积 key 同时过期，用 TTL 加随机值、集群、限流、多级缓存解决
3. **缓存击穿**：热点 key 过期，用互斥锁、逻辑过期、永不过期解决
4. 三种问题本质不同，解决方案也不同，面试和开发中要注意区分

---

## 八、关联笔记

- [[Redis基础篇]]
- [[缓存空对象实现]]

---

## 九、代码变更

- 已实现：`src/main/java/com/hmdp/service/impl/ShopServiceImpl.java` — 缓存空对象防穿透
- 已实现：`src/main/java/com/hmdp/service/impl/ShopServiceImpl.java` — 互斥锁防击穿（queryWithMutex）
- 已实现：`src/main/java/com/hmdp/service/impl/ShopServiceImpl.java` — 逻辑过期防击穿（queryWithLogicalExpire，注释备用）
- 已定义：`src/main/java/com/hmdp/utils/RedisConstants.java` — CACHE_NULL_TTL、LOCK_SHOP_KEY 等常量
