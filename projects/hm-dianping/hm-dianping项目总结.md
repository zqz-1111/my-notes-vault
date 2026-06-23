---
date: '2026-06-23'
tags:
  - hm-dianping
  - Redis
  - 项目总结
  - SpringBoot
---

# hm-dianping 项目总结：从零到一的 Redis 实战

## 一、写在前面

hm-dianping（黑马点评）是一个仿大众点评/美团的本地生活平台，用 8 天时间（6/15 ~ 6/23）完成核心功能开发。这个项目的价值不在于业务本身，而在于**把 Redis 的核心数据结构和典型应用场景全部实战了一遍**。

学完这篇总结，你能掌握：
- Redis 8 大数据结构的实际应用场景
- 缓存三大经典问题的解决方案
- 分布式锁从手写到 Redisson 的演进
- 秒杀场景下的并发安全与异步化
- Feed 流、GEO、Bitmap、HyperLogLog 等进阶用法

## 二、技术栈

| 技术 | 版本 | 作用 |
|---|---|---|
| Java | 8 (JDK 1.8) | 后端语言 |
| Spring Boot | 2.3.12 | 框架 |
| MyBatis-Plus | - | ORM |
| MySQL | localhost:3306 | 数据库 |
| Redis | localhost:6379 | 缓存 + 分布式解决方案 |
| Nginx | 1.18.0 | 前端静态资源 + 图片上传 |

## 三、项目架构

```
Controller → Service → Mapper → MySQL
    ↓
  Redis（缓存/分布式锁/消息队列/统计）
```

统一响应：`Result` DTO（success / errorMsg / data / total）
用户上下文：`UserHolder`（ThreadLocal 存当前登录用户）

## 四、功能模块与 Redis 知识点对照

### 4.1 登录功能

**核心流程：** 发送验证码 → 验证码校验 → 生成 Token → 存入 Redis → 拦截器校验

**Redis 用法：** String 存 Token + 用户信息

```
key: login:token:{随机token}
value: UserDTO 的 JSON
TTL: 36000秒（10小时）
```

**关键设计：**
- 双拦截器模式：第一个拦截器刷新 Token TTL + 放行所有请求；第二个拦截器拦截需要登录的接口
- `UserHolder` 基于 ThreadLocal，请求结束在 `afterCompletion` 中清除

**关联笔记：** [[登录功能完整实现]]

---

### 4.2 Redis 缓存方案

**核心问题：** 数据库扛不住高频读请求，需要缓存分摊压力

**Cache Aside 模式：**
- 读：先查缓存 → 命中返回 → 未命中查 DB → 写入缓存
- 写：先更新 DB → 再删除缓存（而非更新缓存，避免并发不一致）

**三大经典问题：**

| 问题 | 原因 | 解决方案 |
|---|---|---|
| 缓存穿透 | 查询不存在的数据，缓存永远不命中 | 缓存空对象（TTL 2分钟） |
| 缓存雪崩 | 大量 Key 同时过期 | 随机 TTL 打散过期时间 |
| 缓存击穿 | 热点 Key 过期，高并发打到 DB | 互斥锁 / 逻辑过期 |

**互斥锁方案：** 用 Redis 的 `SETNX` 实现分布式锁，只让一个线程去查 DB 并写缓存，其他线程等待重试。

**逻辑过期方案：** 缓存永不过期，但存一个逻辑过期时间。发现过期时开一个新线程去更新缓存，当前请求返回旧数据（不阻塞）。

**CacheClient 封装：** 把缓存穿透、击穿的处理逻辑封装成通用工具类，一个方法搞定。

**关联笔记：** [[Redis缓存三大经典问题]]、[[Redis缓存商铺数据]]、[[缓存模块总结]]、[[CacheClient缓存击穿空值Bug修复]]

---

### 4.3 全局 ID 生成器

**问题：** 数据库自增 ID 在分布式环境下会冲突

**方案：** Redis 自增 + 时间戳 + 序列号位运算拼接

```
64位 Long:
- 高32位：时间戳（秒级，相对于起始时间的偏移）
- 低32位：Redis 自增序列号
```

**关键代码：** `RedisIdWorker.nextId(prefix)` — 每次调用 `INCR` 获取序列号，与时间戳位运算拼接。

**关联笔记：** [[全局ID生成器]]

---

### 4.4 秒杀功能

**基本流程：** 查券 → 判时间 → 判库存 → 一人一单 → CAS 扣库存 → 创建订单

**并发安全演进：**

| 阶段 | 方案 | 问题 |
|---|---|---|
| 1 | synchronized | JVM 锁，集群下失效 |
| 2 | 分布式锁（手写 SETNX） | 可重入、误删、续期问题 |
| 3 | Redisson | 生产级分布式锁 |
| 4 | Lua 脚本 + 异步化 | 性能最优 |

**最终方案（Lua + Redis Stream）：**

```
用户请求 → Lua 脚本原子校验（库存 > 0 && 一人一单）→ 扣库存 → XADD 到 Stream
                                                              ↓
                                          消费者组异步读取 → 创建订单 → XACK 确认
```

**Lua 脚本做的事情（原子性）：**
1. 判断库存是否充足
2. 判断用户是否已下单（Set 判断）
3. 库存 -1
4. 记录用户到已下单 Set

**Redis Stream 消息队列：**
- `XADD` 发送消息
- `XREADGROUP` 消费者组消费（不重复消费）
- `XACK` 确认消费成功
- Pending List 兜底处理异常消息

**关联笔记：** [[秒杀功能实现与并发安全]]、[[一人一单与分布式锁]]、[[异步秒杀实现]]、[[Redis消息队列方案对比]]

---

### 4.5 分布式锁

**手写分布式锁的问题：**
1. 不可重入 — 同一线程无法重复获取
2. 误删别人的锁 — 释放时没验证是否是自己的锁（用 UUID 解决）
3. 原子性问题 — 判断 + 删除不是原子操作（用 Lua 脚本解决）
4. 不会续期 — 业务没执行完锁就过期了

**Redisson 方案：**
- 可重入锁：Hash 结构，`field=线程标识, value=重入次数`
- 加锁/释放：Lua 脚本保证原子性
- 看门狗续期：后台线程每 10 秒检查一次，自动续期 30 秒
- 获取失败：Pub/Sub 订阅锁释放通知，避免忙等待

**主从一致性问题：**
- MultiLock：所有节点都加锁成功才算成功
- RedLock：多数派加锁成功即可（有争议，Kleppmann 批评其不安全）

**关联笔记：** [[Redis分布式锁实现]]、[[Redisson可重入锁原理详解]]、[[Redis分布式锁四大核心缺陷与优化方案]]

---

### 4.6 达人探店

**点赞功能：**
- 数据结构：SortedSet（`ZADD` / `ZREM` / `ZSCORE`）
- 用 Set → SortedSet 改造，支持按点赞时间排序
- Top 5 点赞排行榜：`ZREVRANGE` 取前 5，查 User 信息返回

**关注功能：**
- 关注/取关：MySQL + Redis Set 双写
- 共同关注：`SINTER` 取两个用户关注集合的交集
- 博主主页：查用户信息 + 博主的笔记列表

**Feed 流（推模式）：**
- 发布笔记时，推送到所有粉丝的 Feed（SortedSet）
- key: `feed:{粉丝id}`，score = 笔记时间戳
- 滚动查询：`ZREVRANGEBYSCORE` 分页获取
- 同分值处理：offset 偏移量解决同一秒多条笔记的分页问题

**关联笔记：** [[达人探店点赞功能改造]]、[[好友关注与关注推送]]

---

### 4.7 附近商户

**数据结构：** Redis GEO

**核心命令：**
- `GEOADD`：写入商户经纬度（按类型分 key）
- `GEORADIUS`：查询指定范围内的商户

**实现细节：**
- 按类型分 key 存储：`shop:geo:{typeId}`
- pipeline 批量写入提升性能
- `ORDER BY FIELD` 保序：GEORADIUS 返回的顺序在 MySQL IN 查询后会丢失，需要手动保序
- 前端滚动加载 + limit/skip 分页

**关联笔记：** [[Redis GEO附近商户实现]]

---

### 4.8 签到功能（Bitmap）

**数据结构：** Bitmap（本质是 String 的二进制操作）

**核心命令：**
- `SETBIT`：签到（把某一天的 bit 位设为 1）
- `BITCOUNT`：统计本月签到天数
- `BITFIELD GET`：取出指定位数的 bit，做位运算算连续签到天数

**Key 设计：** `sign:{userId}:{yyyyMM}`，一个月一个 key，每天一个 bit

**连续签到计算：**
1. `BITFIELD GET u{dayOfMonth} 0` — 取出从第 1 天到今天的所有 bit
2. 右移 `(dayOfMonth - 1)` 位 — 让"今天"成为最低位
3. 循环 `(num & 1)` 判断最低位是否为 1，连续则 count++，遇到 0 停止

**踩坑：** 不右移的话，算的是从月初开始的连续天数，不是从今天往回数的。[[前后端对接踩坑记录]]

---

### 4.9 UV 统计（HyperLogLog）

**问题：** 统计每个商铺页面被多少不同用户访问过（UV = Unique Visitor）

**为什么不用 Set：**
- Set 存 100 万 userId → 占几十 MB
- HyperLogLog 存 100 万 → 永远只占 12 KB，误差 0.81%

**核心命令：**
- `PFADD`：记录用户访问（自动去重）
- `PFCOUNT`：统计 UV 数

**Key 设计：** `uv:shop:{shopId}:{yyyyMMdd}`，每天每个商铺一个 HyperLogLog

**实现：** 在查询商铺接口里顺手 `PFADD`，新增一个 `/shop/{id}/uv` 接口用 `PFCOUNT` 查询。

## 五、Redis 数据结构全景图

| 数据结构 | 本项目中的应用 | 核心命令 |
|---|---|---|
| String | 登录 Token、缓存、分布式锁、计数器 | SET/GET/INCR/SETNX |
| Hash | Redisson 可重入锁（存线程标识+重入次数） | HSET/HGET/HINCRBY |
| Set | 关注关系、共同关注、秒杀已下单用户 | SADD/SISMEMBER/SINTER |
| SortedSet | 点赞排行榜、Feed 流 | ZADD/ZREM/ZREVRANGE/ZREVRANGEBYSCORE |
| Bitmap | 签到统计、连续签到计算 | SETBIT/GETBIT/BITCOUNT/BITFIELD |
| HyperLogLog | UV 独立访客统计 | PFADD/PFCOUNT |
| GEO | 附近商户查询 | GEOADD/GEORADIUS |
| Stream | 异步秒杀消息队列 | XADD/XREADGROUP/XACK |

**额外技能：**
- Lua 脚本：秒杀原子校验、分布式锁释放
- Pipeline：GEO 批量写入优化
- Pub/Sub：Redisson 锁释放通知

## 六、未完成功能（后续可做）

以下是项目中还未实现的功能，不影响 Redis 学习效果，后续有时间可以补：

| 功能 | 状态 | 说明 |
|---|---|---|
| 用户登出 | Stub | `POST /user/logout` 返回"功能未完成"，需删 Redis Token + 清 UserHolder |
| 评论功能 | 空壳 | `BlogCommentsController` / Service 全部为空，标准 CRUD 无新知识 |
| 库存扣减 Bug | 待修复 | `VoucherOrderServiceImpl.createVoucherOrder` 中 `success` 未判断，扣库存失败仍会保存订单 |
| ShopType 死代码 | 待清理 | `ShopTypeController.queryTypeList()` 有一行无效的数据库查询 |
| ShopService 注释代码 | 待清理 | `queryWithLogicalExpire` 和 `saveShop2Redis` 旧版注释代码残留 |

## 七、学习路线回顾

```
Day 1 (6/15)  Redis 基础 → 五种数据结构 + Jedis + SpringDataRedis
Day 2 (6/17)  登录 + 缓存 → String Token + Cache Aside + 拦截器
Day 3 (6/18)  缓存进阶 → 穿透/雪崩/击穿 + 互斥锁 + 逻辑过期
Day 4 (6/19)  秒杀基础 → 全局ID + CAS乐观锁 + 超卖问题
Day 5 (6/20)  分布式锁 → synchronized → SETNX → Redisson → RedLock
Day 6 (6/21)  异步秒杀 → Lua脚本 + BlockingQueue + Redis Stream
Day 7 (6/22)  社交功能 → SortedSet点赞 + Set关注 + Feed流 + GEO
Day 8 (6/23)  签到+UV → Bitmap连续签到 + HyperLogLog去重统计
```

## 八、关联笔记

- [[登录功能实现-验证码发送]]
- [[登录功能完整实现]]
- [[Redis缓存商铺数据]]
- [[Redis缓存三大经典问题]]
- [[缓存模块总结]]
- [[CacheClient缓存击穿空值Bug修复]]
- [[全局ID生成器]]
- [[秒杀功能实现与并发安全]]
- [[一人一单与分布式锁]]
- [[Redis分布式锁实现]]
- [[Redisson可重入锁原理详解]]
- [[Redis分布式锁四大核心缺陷与优化方案]]
- [[异步秒杀实现]]
- [[Redis消息队列方案对比]]
- [[达人探店点赞功能改造]]
- [[好友关注与关注推送]]
- [[Redis GEO附近商户实现]]
- [[SpringBoot常用注解大全]]
- [[JVM Target 版本不匹配报错]]
- [[前后端对接踩坑记录]]
