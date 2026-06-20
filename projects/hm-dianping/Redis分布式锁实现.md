---
date: 2026-06-20
tags: [hm-dianping, Redis, 分布式锁, Redisson]
---

# 【hm-dianping】Redis 分布式锁实现｜从手写到 Redisson

## 一、写在前面

1. 为什么学这个 — 之前用 `synchronized` 实现一人一单，但 `synchronized` 只在单机 JVM 内有效，集群环境下不同服务器的锁互不可见，需要分布式锁保证跨 JVM 的互斥访问
2. 学完这篇你能掌握 — 手写 Redis 分布式锁的完整实现 + Lua 脚本原子释放 + Redisson 生产级方案整合

## 二、前置准备

- Redis 已启动（localhost:6379）
- 项目已集成 Spring Data Redis
- 了解 [[Redis基础命令]] 基本操作

## 三、手写分布式锁 — SimpleRedisLock

### 3.1 锁接口定义

先定义一个锁的接口，方便后续扩展：

```java
public interface ILock {
    /**
     * 尝试获取锁
     * @param timeoutSec 锁持有的超时时间，过期后自动释放
     * @return true代表获取锁成功；false代表获取锁失败
     */
    boolean tryLock(long timeoutSec);

    /**
     * 释放锁
     */
    void unlock();
}
```

### 3.2 核心实现

```java
public class SimpleRedisLock implements ILock {

    private StringRedisTemplate stringRedisTemplate;
    private String name;

    /**
     * key前缀
     */
    public static final String KEY_PREFIX = "lock:";
    /**
     * ID前缀（UUID标识不同JVM实例，防跨进程误删）
     */
    public static final String ID_PREFIX = UUID.randomUUID().toString(true) + "-";

    /**
     * 释放锁的Lua脚本（类加载时初始化，避免每次调用都编译）
     */
    private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
    static {
        UNLOCK_SCRIPT = new DefaultRedisScript<>();
        UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
        UNLOCK_SCRIPT.setResultType(Long.class);
    }

    public SimpleRedisLock(StringRedisTemplate stringRedisTemplate, String name) {
        this.stringRedisTemplate = stringRedisTemplate;
        this.name = name;
    }

    @Override
    public boolean tryLock(long timeoutSec) {
        String threadId = ID_PREFIX + Thread.currentThread().getId() + "";
        // SET lock:name id EX timeoutSec NX（原子操作）
        Boolean result = stringRedisTemplate.opsForValue()
                .setIfAbsent(KEY_PREFIX + name, threadId, timeoutSec, TimeUnit.SECONDS);
        return Boolean.TRUE.equals(result);
    }

    @Override
    public void unlock() {
        // 执行Lua脚本，原子性地判断+删除
        stringRedisTemplate.execute(
                UNLOCK_SCRIPT,
                Collections.singletonList(KEY_PREFIX + name),  // KEYS[1]
                ID_PREFIX + Thread.currentThread().getId()     // ARGV[1]
        );
    }
}
```

### 3.3 Lua 脚本 — unlock.lua

在 `src/main/resources/unlock.lua` 创建：

```lua
-- KEYS[1] = 锁的key，ARGV[1] = 当前线程标识
-- 判断锁的标识是否与当前线程一致
if redis.call('GET', KEYS[1]) == ARGV[1] then
    -- 一致，释放锁
    return redis.call('DEL', KEYS[1])
end
-- 不一致，不释放
return 0
```

> **踩坑点：为什么用 Lua 脚本？**
> 如果用 Java 代码先 GET 判断再 DEL 删除，两步操作之间有时间差，可能出现：
> 1. 线程 A GET 判断锁是自己的
> 2. 锁刚好过期，线程 B 拿到锁
> 3. 线程 A DEL 删除了线程 B 的锁 ← 误删！
> 
> Lua 脚本在 Redis 中是单线程原子执行的，中间不会被打断。

### 3.4 业务中使用锁

```java
// seckillVoucher 方法中
Long userId = UserHolder.getUser().getId();
SimpleRedisLock lock = new SimpleRedisLock(stringRedisTemplate, "order:" + userId);
boolean isLock = lock.tryLock(1200);  // 1200秒超时
if (!isLock) {
    return Result.fail("一人只能下一单");
}
try {
    IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
    return proxy.createVoucherOrder(userId, voucherId);
} finally {
    lock.unlock();
}
```

> **踩坑点：为什么用构造函数而不是 @Resource 注入？**
> 因为 `name` 参数每个用户都不一样（`order:用户ID`），不能用单例 Bean。
> `@Resource` 是注入 Spring Bean 用的，`String` 不是 Bean，注入会失败。
> 这种参数不固定、用完就丢的工具类，构造函数 + 手动 `new` 最合适。

### 3.5 手写锁的局限

| 问题 | 说明 |
|---|---|
| 锁误删（已解决） | UUID + 线程标识，Lua 脚本原子校验 |
| 不可重入 | 同一线程无法重复获取同一把锁 |
| 不可重试 | 拿不到直接返回 false，没有等待机制 |
| 无自动续期 | 业务执行超时，锁自动过期，其他线程能进来 |

这四个问题就是生产级分布式锁必须解决的，详见 [[Redis分布式锁四大核心缺陷与优化方案]]。

## 四、Redisson 生产级方案

### 4.1 添加依赖

```xml
<!--pom.xml-->
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.13.6</version>
</dependency>
```

> **版本选择**：Spring Boot 2.3.x + Java 8 用 Redisson 3.x，Spring Boot 3.x 用 Redisson 3.20+

### 4.2 配置类

```java
@Configuration
public class RedissonConfig {

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        // 单点模式，集群用 config.useClusterServers()
        config.useSingleServer().setAddress("redis://127.0.0.1:6379");
        return Redisson.create(config);
    }
}
```

> **踩坑点：Redis 地址协议**
> 必须用 `redis://` 不是 `http://`，集群模式用 `redis://ip:port`

### 4.3 使用 Redisson 锁

```java
@Resource
private RedissonClient redissonClient;

// 获取锁
RLock lock = redissonClient.getLock("lock:order:" + userId);

// 尝试获取锁（无参：非阻塞，拿不到立刻返回 false）
boolean isLock = lock.tryLock();

if (!isLock) {
    return Result.fail("一人只能下一单");
}
try {
    IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
    return proxy.createVoucherOrder(userId, voucherId);
} finally {
    lock.unlock();
}
```

### 4.4 Redisson vs 手写对比

| 特性 | 手写 SimpleRedisLock | Redisson RLock |
|---|---|---|
| 互斥 | ✅ SETNX | ✅ SETNX |
| 原子释放 | ✅ Lua 脚本 | ✅ Lua 脚本 |
| 防误删 | ✅ UUID + 线程ID | ✅ 内置标识 |
| 可重入 | ❌ | ✅ Redis Hash 记录重入次数 |
| 可重试 | ❌ | ✅ 自旋 / 发布订阅 |
| 自动续期 | ❌ | ✅ 看门狗（30秒过期，每10秒续一次） |
| 主从一致性 | ❌ | ✅ 红锁（RedLock） |

## 五、踩坑避坑指南

### 逻辑坑

**坑 1：unlock 的逻辑运算符**
- 问题：`if (currentThreadFlag != null || ...)` 用了 `||`，`currentThreadFlag` 永远不为 null，条件永远为真
- 后果：任何线程都能释放锁，防误删失效
- 正确做法：改成 `&&`

**坑 2：createVoucherOrder 参数不一致**
- 问题：接口定义 `createVoucherOrder(Long voucherId)`，实现改成两个参数但接口没同步
- 后果：编译报错 "必须声明为抽象或实现抽象方法"
- 正确做法：接口和实现的方法签名要保持一致

**坑 3：ThreadLocalUtls 拼写错误**
- 问题：`ThreadLocalUtls` 类名拼错
- 后果：编译报错找不到类
- 正确做法：用 `UserHolder.getUser()`

### 报错坑

**坑 4：@Resource 注入 String 失败**
- 报错信息：注入失败或运行时 NPE
- 原因：`String` 不是 Spring Bean，`@Resource` 无法注入
- 解决方案：改用构造函数传参

## 六、效果验证

1. 启动项目，确认日志出现 `Redisson 3.13.6`
2. 登录获取 Token
3. 请求 `POST /voucher-order/seckill/{voucherId}`
4. 第一次请求成功返回订单 ID
5. 第二次请求返回 "一人只能下一单"

## 七、全文总结

- 分布式锁核心：SETNX 互斥 + 过期时间防死锁 + UUID 防误删 + Lua 脚本原子释放
- 手写锁能解决基本问题，但不可重入、不可重试、无续期、无主从一致性
- Redisson 是生产级方案，开箱即用，四大缺陷全部解决
- 先学原理再用工具，面试能讲清楚底层实现

## 八、关联笔记

- [[Redis分布式锁四大核心缺陷与优化方案]]
- [[一人一单实现]]

## 九、代码变更

- 新增：`src/main/java/com/hmdp/utils/ILock.java` — 锁接口
- 新增：`src/main/java/com/hmdp/utils/SimpleRedisLock.java` — 手写分布式锁实现
- 新增：`src/main/resources/unlock.lua` — Lua 脚本原子释放锁
- 新增：`src/main/java/com/hmdp/config/RedissonConfig.java` — Redisson 配置类
- 修改：`pom.xml` — 添加 redisson-spring-boot-starter 依赖
- 修改：`src/main/java/com/hmdp/service/impl/VoucherOrderServiceImpl.java` — 使用 Redisson 替代手写锁
