---
date: '2026-06-19'
tags:
  - hm-dianping
  - Redis
  - 分布式ID
  - 雪花算法
---

# 【Redis】全局 ID 生成器｜RedisIdWorker 实现原理

## 一、写在前面

1. 为什么学这个 — 秒杀下单需要生成订单 ID，数据库自增 ID 在分布式场景下有冲突、性能、安全等问题，需要一个全局唯一的 ID 生成方案
2. 学完这篇你能掌握 — 分布式 ID 的常见方案对比、Redis 自增 + 时间戳拼接的实现原理、位运算的精髓

## 二、前置准备

- 了解 Redis 的 `INCR` 命令（原子自增）
- 了解 Java 位运算（`<<` 左移、`|` 按位或）
- 项目环境：Java 8 + Spring Boot + Redis

## 三、为什么数据库自增 ID 不够用？

单库的时候 `AUTO_INCREMENT` 挺好用，但分布式/高并发场景下有问题：

| 问题 | 说明 |
|---|---|
| **ID 可预测** | 用户能猜到下一个订单号，有安全风险 |
| **性能瓶颈** | 高并发时每次都等数据库分配 ID，扛不住 |
| **分库分表冲突** | 多个库各自自增，id 会重复 |

## 四、常见方案对比

| 方案 | 原理 | 优点 | 缺点 | 谁在用 |
|---|---|---|---|---|
| **UUID** | 128 位随机字符串 | 简单、无需依赖 | 无序（索引差）、太长 | 通用场景 |
| **数据库自增** | `AUTO_INCREMENT` | 简单、有序 | 分布式冲突、性能瓶颈 | 单体应用 |
| **雪花算法** | 时间戳 + 机器号 + 序列号 | 高性能、趋势递增 | 需解决时钟回拨 | 美团、百度 |
| **Redis 自增** | `INCR` 原子操作 | 简单、有序、高性能 | 依赖 Redis | 中小规模 |

**你项目的做法：Redis 自增 + 时间戳拼接**，兼具两者优点。

## 五、RedisIdWorker 实现详解

### 5.1 整体思路

把一个 64 位的 long 拆成两部分：

```
┌──────────────────────┬──────────────────────┐
│   高 32 位：时间戳     │   低 32 位：序列号     │
└──────────────────────┴──────────────────────┘
```

- **高 32 位**：当前时间（秒）减去一个起点，表示"距离 2022-01-01 过了多少秒"
- **低 32 位**：Redis `INCR` 生成的自增序列号，每天一个新 key，从 1 开始

### 5.2 完整代码

```java
@Component
public class RedisIdWorker {

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    /**
     * 开始时间戳（2022-01-01 00:00:00 UTC）
     */
    private static final long BEGIN_TIMESTAMP = 1640995200;

    /**
     * 序列号占用的位数
     */
    private static final int COUNT_BITS = 32;

    /**
     * 生成分布式 ID
     * @param keyPrefix 业务前缀，如 "order"
     * @return 全局唯一 ID
     */
    public long nextId(String keyPrefix) {
        // 1、生成时间戳
        LocalDateTime now = LocalDateTime.now();
        long nowSecond = now.toEpochSecond(ZoneOffset.UTC);
        long timestamp = nowSecond - BEGIN_TIMESTAMP;

        // 2、生成序列号（按天分 key，每天从 1 开始）
        String date = now.format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        Long count = stringRedisTemplate.opsForValue()
            .increment(ID_PREFIX + keyPrefix + ":" + date);

        // 3、拼接并返回：时间戳左移 32 位，再用按位或填入序列号
        return timestamp << COUNT_BITS | count;
    }
}
```

### 5.3 第一步：生成时间戳

```java
LocalDateTime now = LocalDateTime.now();
long nowSecond = now.toEpochSecond(ZoneOffset.UTC);
long timestamp = nowSecond - BEGIN_TIMESTAMP;
```

- `now.toEpochSecond(ZoneOffset.UTC)` — 当前时间转成 UTC 秒数
- 减去 `BEGIN_TIMESTAMP`（2022-01-01 的秒数）— 让数字变小，省位数

```
当前：2026-06-19 UTC
nowSecond = 1781956200
timestamp = 1781956200 - 1640995200 = 140961000
```

**为什么减起点？** 如果直接用 `1781956200`，二进制要 31 位；减完后 `140961000` 只要 27 位，省出来的位给序列号用。

### 5.4 第二步：生成序列号

```java
String date = now.format(DateTimeFormatter.ofPattern("yyyyMMdd"));
Long count = stringRedisTemplate.opsForValue()
    .increment(ID_PREFIX + keyPrefix + ":" + date);
```

- Redis key 格式：`id:order:20260619`
- `increment` = Redis 的 `INCR` 命令，原子操作，每次 +1
- 第 1 次调用返回 1，第 2 次返回 2……

**为什么按天分 key？**

```
不分 key：id:order 永续增长 → 几年后数字巨大，32 位可能溢出
按天分 key：每天最多 2^32 ≈ 42 亿个 ID，永远安全
额外好处：每天的 key 还能统计当天订单量
```

### 5.5 第三步：位运算拼接

```java
return timestamp << COUNT_BITS | count;
```

这行是精髓，拆开看：

```
timestamp = 140961000（二进制：1000011001110001010010001000）
count     = 42       （二进制：101010）

步骤 1：左移 32 位（timestamp 后面补 32 个 0）
  1000011001110001010010001000 00000000000000000000000000000000

步骤 2：按位或（count 填进低 32 位）
  1000011001110001010010001000 00000000000000000000000000101010
  └────── 时间戳部分 ───────┘ └────── 序列号部分 ───────┘
```

**为什么用位运算而不是字符串拼接？**
- 位运算直接操作二进制，性能比字符串拼接高一个数量级
- 最终结果是一个 long 类型，占 8 字节；字符串拼接会占 20+ 字节

## 六、生成的 ID 长什么样？

```
第 1 个订单：timestamp=140961000, count=1  → 606347276289
第 2 个订单：timestamp=140961000, count=2  → 606347276290
第 3 个订单：timestamp=140961001, count=1  → 606347280385（下一秒了）
```

**特性：**
- **全局唯一**：不同秒不会重复，同秒内靠序列号区分
- **趋势递增**：时间在走，ID 整体越来越大，数据库 B+ 树索引友好
- **可反推时间**：`id >> 32 + BEGIN_TIMESTAMP` 就能算出生成时间
- **按天重置**：序列号每天归零，防溢出

## 七、调用方式

```java
@Resource
private RedisIdWorker redisIdWorker;

// 生成订单 ID
long orderId = redisIdWorker.nextId("order");
```

Redis 里的 key：`id:order:20260619`，每次调用 `INCR`，返回 count，再和时间戳拼接。

## 八、踩坑避坑

### 坑 1：ID_PREFIX 常量未定义

- 报错信息：`无法解析符号 'ID_PREFIX'`
- 原因：`RedisIdWorker` 里用了 `ID_PREFIX`，但 `RedisConstants.java` 没定义
- 解决方案：在 `RedisConstants.java` 加 `public static final String ID_PREFIX = "id:";`

### 坑 2：时区问题

- 问题：`toEpochSecond(ZoneOffset.UTC)` 用的是 UTC 时间，不是本地时间
- 影响：同一天内 UTC 和本地时间可能差几个小时，但不影响 ID 唯一性
- 注意：如果业务需要按"本地日期"分 key，需要改成 `ZoneOffset.ofHours(8)`

## 九、全文总结

1. 分布式 ID 的核心需求：**全局唯一、趋势递增、高性能**
2. RedisIdWorker 的方案：**时间戳（高 32 位）+ Redis 自增序列号（低 32 位）**
3. 按天分 key 的好处：**序列号不溢出 + 可统计当天订单量**
4. 位运算拼接的好处：**性能高、占用空间小**

## 十、关联笔记

- [[缓存模块总结]] — Redis 在缓存层面的应用
- [[Redis缓存三大经典问题]] — 缓存穿透、雪崩、击穿

## 十一、代码变更

- 修改：`src/main/java/com/hmdp/utils/RedisConstants.java` — 新增 `ID_PREFIX` 常量
- 修改：`src/main/java/com/hmdp/utils/RedisIdWorker.java` — 添加 `ID_PREFIX` 静态导入
