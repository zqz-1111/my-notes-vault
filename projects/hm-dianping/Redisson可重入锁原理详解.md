---
date: 2026-06-20
tags: [hm-dianping, Redis, Redisson, 分布式锁, 面试]
---

# 【Redisson】可重入锁原理详解｜源码级拆解

## 一、写在前面

1. 为什么学这个 — 之前整合了 Redisson 替代手写 SimpleRedisLock，但只知道"好用"，不知道"怎么实现的"。面试问到原理就懵。
2. 学完你能掌握 — Redisson 可重入锁的 Hash 数据结构、加锁/释放 Lua 脚本源码、看门狗续期机制、主从一致性问题及解决方案。

## 二、先看手写锁的致命缺陷

之前手写的 `SimpleRedisLock` 用 `SET NX` 加锁：

```java
// tryLock 方法
Boolean result = stringRedisTemplate.opsForValue()
    .setIfAbsent(KEY_PREFIX + name, threadId, timeoutSec, TimeUnit.SECONDS);
```

**问题：同一线程二次加锁会死锁。**

```java
public void methodA() {
    lock.tryLock();  // SET lock:order:10 uuid-1-1 NX OK ✅
    methodB();       // 调用 methodB
    lock.unlock();
}

public void methodB() {
    lock.tryLock();  // SET lock:order:10 uuid-1-1 NX ❌ key已存在，失败！
    // 返回 false → 认为获取锁失败 → 死锁
}
```

**本质原因：SET NX 不区分"别人持有"和"自己持有"。**

## 三、Redisson 的数据结构：Hash

Redisson 不用 String，用 **Redis Hash**：

```
KEY:   lock:order:10          （锁的名称）
FIELD: 8abf3a91-1-14          （UUID + 线程ID，客户端唯一标识）
VALUE: 1                       （重入次数）
```

| 对比项 | 手写 SimpleRedisLock | Redisson RLock |
|---|---|---|
| 数据结构 | `String: lock:order:10 → threadId` | `Hash: lock:order:10 → {threadId → count}` |
| 能否重入 | ❌ SET NX 不允许重复 key | ✅ 判断 field 是否是自己，是就 count++ |
| 超时方式 | 固定 TTL | 看门狗自动续期 |

## 四、加锁 Lua 脚本源码解析

Redisson 加锁的核心 Lua 脚本（简化版）：

```lua
-- KEYS[1] = 锁的名称，如 "lock:order:10"
-- ARGV[1] = 锁超时时间（毫秒）
-- ARGV[2] = 线程唯一标识（UUID:threadId）

-- 1. 判断锁是否存在
if (redis.call('EXISTS', KEYS[1]) == 0) then
    -- 不存在，首次加锁
    redis.call('HSET', KEYS[1], ARGV[2], 1)        -- HSET lock:order:10 threadId 1
    redis.call('PEXPIRE', KEYS[1], ARGV[1])         -- 设置过期时间
    return nil  -- 加锁成功
end

-- 2. 锁存在，判断是不是自己持有的（可重入的关键）
if (redis.call('HEXISTS', KEYS[1], ARGV[2]) == 1) then
    -- 是自己持有的，重入次数 +1
    redis.call('HINCRBY', KEYS[1], ARGV[2], 1)
    redis.call('PEXPIRE', KEYS[1], ARGV[1])         -- 续期
    return nil  -- 加锁成功（重入）
end

-- 3. 锁存在，且不是自己持有的 → 加锁失败
return redis.call('PTTL', KEYS[1])  -- 返回锁的剩余时间
```

**流程图：**

```
tryLock()
    │
    ▼
┌─────────────────┐
│  EXISTS key ?   │
└────────┬────────┘
    No   │   Yes
    │    │
    ▼    ▼
┌──────┐ ┌──────────────────┐
│首次   │ │ HEXISTS field?   │
│加锁   │ └────────┬─────────┘
│HSET 1│     Yes   │   No
│续期   │     │     │
└──────┘     ▼     ▼
           ┌────┐ ┌──────────┐
           │重入  │ │加锁失败   │
           │+1   │ │返回剩余TTL│
           │续期  │ └──────────┘
           └────┘
```

## 五、释放锁 Lua 脚本源码解析

```lua
-- KEYS[1] = 锁的名称
-- ARGV[1] = 锁超时时间（毫秒）
-- ARGV[2] = 线程唯一标识

-- 1. 判断锁是不是自己持有的
if (redis.call('HEXISTS', KEYS[1], ARGV[2]) == 0) then
    -- 不是自己持有的，不能释放！
    return nil
end

-- 2. 是自己持有的，重入次数 -1
local counter = redis.call('HINCRBY', KEYS[1], ARGV[2], -1)

-- 3. 判断重入次数是否归零
if (counter > 0) then
    -- 还有重入，只续期不删除
    redis.call('PEXPIRE', KEYS[1], ARGV[1])
    return 0   -- 表示还有重入锁未释放
else
    -- 重入次数为 0，彻底删除锁
    redis.call('DEL', KEYS[1])
    return 1   -- 表示锁彻底释放
end
```

**对应 methodA → methodB 的场景：**

```
methodA 加锁:  HSET lock:order:10 threadId → 1    (count=1)
  └→ methodB 加锁: HINCRBY → 2                    (count=2，重入)
  └→ methodB 释放: HINCRBY -1 → 1                  (count=1，还有重入)
methodA 释放:  HINCRBY -1 → 0 → DEL               (count=0，彻底释放)
```

## 六、看门狗（Watchdog）机制

这是 Redisson 最精妙的设计。

### 6.1 为什么需要看门狗？

手写锁用固定 TTL，有两个问题：

| 问题 | 手写锁 | Redisson 看门狗 |
|---|---|---|
| 业务没执行完，锁过期了 | ❌ 会超卖/重复下单 | ✅ 自动续期 |
| 锁设置太长，宕机后别人等太久 | ❌ 要等 TTL 过期 | ✅ 最多等 30s |

### 6.2 看门狗原理

```
加锁成功
    │
    ▼
┌─────────────────────────────────────┐
│ 启动后台调度线程（Netty TimerTask）    │
│ 每隔 lockWatchdogTimeout / 3        │
│ （默认 30s / 3 = 10s）               │
│ 检查锁是否还被当前线程持有              │
│ 如果是 → 重置 TTL 为 30s             │
│ 如果否 → 停止续期                     │
└─────────────────────────────────────┘
```

**源码级流程：**

```java
// 加锁成功后，启动看门狗
private void scheduleExpirationRenewal(long threadId) {
    Timeout task = commandExecutor.getConnectionManager()
        .newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) {
                // 每 10s 执行一次
                // 续期：重置 TTL 为 30s
                renewExpiration();  // → Lua: PEXPIRE key 30000
                
                // 递归调度下一次
                scheduleExpirationRenewal(threadId);
            }
        }, lockWatchdogTimeout / 3, TimeUnit.MILLISECONDS);  // 10s
}
```

### 6.3 关键点

- **看门狗只在不指定 leaseTime 时生效**：`lock.tryLock()` → 有看门狗；`lock.tryLock(10, TimeUnit.SECONDS)` → 没有
- **锁释放后看门狗自动停止**：unlock 时取消定时任务
- **宕机后最多等 30s**：看门狗没了，TTL 自然过期
- **兜底机制**：看门狗最多续期 30 分钟（lockWatchdogTimeout × lockWatchdogTimeoutMultiplier），避免死循环永久持锁

### 6.4 看门狗时间线

```
0s     加锁成功，启动看门狗，TTL=30s
10s    第一次续期，重置 TTL=30s
20s    第二次续期，重置 TTL=30s
30s    第三次续期，重置 TTL=30s
...
业务完成  停止续期，释放锁
```

## 七、重试机制：自旋 + Pub/Sub

Redisson `tryLock()` 的等待机制不是信号量，是 **自旋 + 订阅/释放通知**：

```java
// Redisson 源码简化逻辑
while (true) {
    // 1. 尝试加锁
    Long ttl = tryAcquire();
    if (ttl == null) return true;  // 成功
    
    // 2. 加锁失败，订阅锁的释放消息（不是忙等轮询）
    subscribe(threadId);  // 阻塞等待
    
    // 3. 收到释放通知后，回到 while 循环再次尝试
}
```

| 方式 | 原理 | 效率 |
|---|---|---|
| 忙等轮询 | while 循环 + sleep | CPU 浪费，Redis 压力大 |
| Pub/Sub（Redisson 实际用的） | 订阅锁 key 的释放事件，被动唤醒 | 高效，不用轮询 |

## 八、主从一致性问题

### 8.1 问题场景

Redis 主从复制是异步的：

```
T1  客户端A → 主节点加锁 SET lock:order:10 uuid-A NX OK
T2  主节点 → 从节点（异步复制，还没同步到）
T3  主节点宕机！
T4  哨兵选举从节点为新主（锁数据丢失）
T5  客户端B → 新主节点加锁 SET lock:order:10 uuid-B NX OK
T6  客户端A 和 客户端B 同时持有锁 → 💥 超卖
```

### 8.2 解决方案对比

| 方案 | 原理 | 优点 | 缺点 |
|---|---|---|---|
| **MultiLock** | 多个独立 Redis 节点，**全部**加锁成功才算成功 | 强一致性 | 部署成本高，性能差 |
| **RedLock** | 多个独立 Redis 节点，**过半**加锁成功就算成功 | 平衡一致性和性能 | 有争议（Kleppmann 质疑） |

### 8.3 MultiLock 代码

```java
// 多个完全独立的 Redis 节点（不是主从，是独立实例）
Config config1 = new Config();
config1.useSingleServer().setAddress("redis://192.168.1.101:6379");
RedissonClient client1 = Redisson.create(config1);

Config config2 = new Config();
config2.useSingleServer().setAddress("redis://192.168.1.102:6379");
RedissonClient client2 = Redisson.create(config2);

Config config3 = new Config();
config3.useSingleServer().setAddress("redis://192.168.1.103:6379");
RedissonClient client3 = Redisson.create(config3);

// 获取每个节点的锁
RLock lock1 = client1.getLock("lock:order:10");
RLock lock2 = client2.getLock("lock:order:10");
RLock lock3 = client3.getLock("lock:order:10");

// 创建 MultiLock（所有节点都要加锁成功）
RedissonMultiLock multiLock = new RedissonMultiLock(lock1, lock2, lock3);

boolean isLock = multiLock.tryLock(10, 30, TimeUnit.SECONDS);
```

### 8.4 RedLock 代码

```java
// 和 MultiLock 类似，但用 RedissonRedLock
RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);

// 过半节点加锁成功即可（3个节点中2个成功）
boolean isLock = redLock.tryLock(10, 30, TimeUnit.SECONDS);
```

### 8.5 RedLock 争议（面试加分项）

**Martin Kleppmann（《数据密集型应用系统设计》作者） vs Antirez（Redis 作者）**

| 观点 | Kleppmann | Antirez |
|---|---|---|
| 核心论点 | RedLock 依赖时钟同步，不安全 | 时钟漂移可以忽略，算法正确 |
| 问题 | GC 停顿 / 网络延迟可能导致锁过期但客户端不知道 | 可以用 fencing token 解决 |
| 结论 | 不推荐 RedLock 做强一致场景 | RedLock 在工程实践中够用 |

## 九、三代锁演进总结

| 锁类型 | 核心特性 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|---|
| 不可重入锁 | SETNX + 超时 + 标识校验 | 实现简单、性能高 | 不可重入、无重试、超时风险 | 简单场景、低并发 |
| Redisson 可重入锁 | Hash + 看门狗 + Pub/Sub | 支持重入、自动续期、易用 | 主从架构下有锁失效风险 | 大多数业务场景 |
| Redisson MultiLock/RedLock | 多节点 + 可重入 | 强一致性、无锁失效风险 | 性能开销高、部署复杂 | 高一致性要求的核心业务 |

## 十、项目中的选择

当前 hm-dianping 秒杀下单用的是 **普通 RLock**：

```java
RLock lock = redissonClient.getLock("lock:order:" + userId);
boolean isLock = lock.tryLock();
```

**不需要换 MultiLock/RedLock**，原因：
1. 秒杀场景允许极端情况下偶尔多卖一单（业务可接受）
2. 数据库乐观锁（`stock > 0`）是最后一道兜底
3. MultiLock 部署成本高（需要多个独立 Redis 实例）

**真正需要强一致的场景：** 金融转账、核心库存扣减（这时可能需要分布式事务或数据库锁）。

## 十一、面试追问

**Q：Redisson 可重入锁的原理？**
> 用 Redis Hash 结构，key 是锁名，field 是客户端标识（UUID+线程ID），value 是重入次数。加锁时判断 field 是否存在，不存在就 HSET 设为 1，存在就 HINCRBY +1。释放时 HINCRBY -1，减到 0 就 DEL。加锁和释放都用 Lua 脚本保证原子性。

**Q：看门狗怎么实现的？**
> 加锁成功后启动一个 Netty 的定时任务，每隔 lockWatchdogTimeout/3（默认10秒）检查锁是否还被持有，是就重置 TTL。指定 leaseTime 时看门狗不生效。

**Q：为什么用 Hash 而不是 String？**
> String 存不了重入次数。如果用 String，第一次加锁 `SET key threadId`，重入时无法区分"别人持有"和"自己持有"，也无法记录重入了几次。

**Q：主从一致性怎么解决？**
> 两种方案：MultiLock 要求所有节点都加锁成功，RedLock 要求过半节点成功。但都有局限，Martin Kleppmann 认为 RedLock 不能解决 GC 停顿和时钟跳跃问题。实际项目中，秒杀这种场景用普通 RLock + 数据库乐观锁兜底就够了。

## 十二、关联笔记

- [[Redis分布式锁实现]] — 手写锁 + Redisson 整合代码
- [[Redis分布式锁四大核心缺陷与优化方案]] — 四大缺陷概述
- [[一人一单与分布式锁]] — synchronized 到分布式锁的演进
