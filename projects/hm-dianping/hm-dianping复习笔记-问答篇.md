---
date: '2026-06-23'
tags:
  - hm-dianping
  - Redis
  - 复习
  - 面试
---

# hm-dianping 复习笔记：问答篇

## 写在前面

通过问答形式复习 hm-dianping 项目的核心知识点，覆盖 DTO、拦截器、缓存、锁、秒杀等模块。适合面试前快速回顾。

---

## 一、DTO 层

### 1.1 DTO 包的 4 个类分别干嘛？

| 类 | 作用 | 示例 |
|---|---|---|
| `Result` | 统一 API 响应封装 | `{ success: true, data: ..., errorMsg: null }` |
| `LoginFormDTO` | 登录表单接收 | 前端传来的 phone、code、password |
| `UserDTO` | 用户信息脱敏 | 只有 id、nickName、icon，不暴露密码 |
| `ScrollResult` | Feed 流滚动分页 | list + minTime + offset |

### 1.2 为什么用 UserDTO 而不是 User？

防止泄露敏感信息（密码、手机号等），只暴露安全字段给前端。

---

## 二、UserHolder 与 ThreadLocal

### 2.1 UserHolder 是干嘛的？

基于 `ThreadLocal<UserDTO>` 实现的**请求级用户上下文**。

```
拦截器查 Redis → UserHolder.saveUser() 存进去
Controller/Service → UserHolder.getUser() 取出来用
请求结束 → UserHolder.removeUser() 清掉
```

### 2.2 ThreadLocal 是什么？

**线程的私人储物柜**，每个线程有独立的变量副本，互不干扰。

- 存值：`tl.set(value)`
- 取值：`tl.get()`
- 清除：`tl.remove()`（必须清，防止内存泄漏）

### 2.3 为什么用 ThreadLocal？

Tomcat 多线程处理请求，用 ThreadLocal 保证每个请求拿到的是自己的用户信息，不会串。

### 2.4 UserHolder 什么时候被清空？

拦截器的 `afterCompletion` 方法中清空。必须清，因为 Tomcat 线程池会复用线程，不清的话下一个请求会拿到上一个用户的脏数据。

### 2.5 请求前数据在哪？

在 **Redis** 里。UserHolder 只是请求级别的临时缓存，每次请求都从 Redis 查一次存进去，用完就清。

### 2.6 为什么要加 UserHolder 中间层？

**不是为了快，是为了不重复写代码。** 拦截器查一次存起来，后面几十个 Controller 直接 `getUser()` 就行，不用每个方法都写 Redis 查询逻辑。

---

## 三、登录与拦截器

### 3.1 session.setAttribute("code", code) 是干嘛的？

把验证码存到 HttpSession 里，登录时取出来比对。但项目实际用 **Redis** 存验证码，因为 Session 有集群不共享、重启丢失等问题。

### 3.2 构造器注入和 @Autowired 的区别？

| | 构造器注入 | @Autowired |
|---|---|---|
| 谁创建对象 | 手动 new | Spring 自动创建 |
| 需要 @Component | 不需要 | 必须 |
| 适合场景 | 不归 Spring 管理的类 | 归 Spring 管理的类 |

拦截器用构造器注入，因为是手动 new 的，不在 Spring 容器里。

### 3.3 MvcConfig 和 LoginInterceptor 的关系？

```
MvcConfig = 保安队长（制定规则：哪些路径要查、哪些放行）
LoginInterceptor = 保安（执行规则：查 Token、存 UserHolder、刷新 TTL）
```

### 3.4 为什么要刷新 Token TTL？

防止用户正在操作时被踢下线。每次请求刷新 25 天，用户活跃就续期，停止操作 25 天后才失效。

---

## 四、缓存相关

### 4.1 Cache Aside 模式是什么？

**旁路缓存**，最常用的缓存模式：
- 读：先缓存 → 没有查 DB → 写入缓存
- 写：先更新 DB → 再删除缓存（不是更新缓存）

### 4.2 为什么写的时候删缓存而不是更新缓存？

避免并发不一致。两个线程同时更新，可能缓存和 DB 的值不一致。删除缓存让下次读的时候从 DB 查最新值。

### 4.3 逻辑过期是什么？

缓存**永不过期**（Redis key 不设 TTL），但在 value 里存一个**逻辑过期时间**。读的时候自己判断：
- 没过期 → 直接返回
- 过期了 → 返回旧数据 + 开新线程去更新缓存

解决**缓存击穿**：热点 key 过期时不会打到 DB。

### 4.4 项目里有缓存雪崩吗？

没有专门处理，但用**逻辑过期**规避了（缓存永不过期，不存在同时过期的情况）。如果用普通 TTL，需要加随机 TTL 打散过期时间。

### 4.5 互斥锁是什么？和悲观锁什么关系？

**互斥锁就是悲观锁的一种**。"我先锁住，别人等着"。

项目里用 Redis SETNX 实现，解决缓存击穿——只让一个请求去查 DB 重建缓存，其他请求返回旧数据。

---

## 五、乐观锁与悲观锁

### 5.1 乐观锁是什么？

**不加锁**，修改时才检查有没有人改过。没人动就成功，有人动就重试。

项目扣库存：`WHERE stock > 0`，执行 UPDATE 时才校验。

### 5.2 乐观锁的"提交"是指什么？

指**执行 UPDATE 的时候**。读数据时不加锁，等到真正写入数据库那一刻，才用 WHERE 条件检查数据有没有被别人改过。

### 5.3 项目里用了哪些锁？

| 场景 | 锁类型 | 实现 |
|---|---|---|
| 扣库存 | 乐观锁（CAS） | `WHERE stock > 0` |
| 缓存击穿 | 互斥锁（悲观锁） | Redis SETNX |
| 一人一单 | 分布式锁（悲观锁） | Redisson |

---

## 六、一人一单与分布式锁

### 6.1 synchronized + intern() 是什么意思？

```java
synchronized (userId.toString().intern()) {
    // 同一用户串行执行
}
```

synchronized 锁的是**对象地址**，不是值。`intern()` 保证同一个 userId 拿到常量池里同一个字符串对象，这样同一用户才能锁住同一把锁。

### 6.2 一个项目一个 JVM 吗？

对，一个启动类 = 一个 JVM 进程。开发时就一个，生产环境可能部署多个（集群），这时候 JVM 锁就不够用了，得用分布式锁。

### 6.3 看门狗机制是什么？

Redisson 内置的**自动续期**机制：
- 锁默认 30 秒过期
- 看门狗每 10 秒检查一次（过期时间的 1/3）
- 业务还在跑就续期，业务结束就不续了

防止业务没执行完锁就过期。用在**一人一单**流程里。

---

## 七、数据结构与工具类

### 7.1 User::getPhone 是什么写法？

Java 8 **方法引用**，等价于 `user -> user.getPhone()`。MyBatis-Plus 用它指定字段，写错了编译器会报错，比字符串安全。

### 7.2 BeanUtil 是什么？

Hutool 工具包的对象属性拷贝工具：
- `BeanUtil.copyProperties(user, UserDTO.class)` — 对象转对象
- `BeanUtil.beanToMap(userDTO)` — 对象转 Map（存 Redis Hash 用）

### 7.3 Map\<String, Object\> 是什么？

key 是 String（字段名），value 是 Object（任意类型）。最通用的 Map 写法。

### 7.4 BeanUtil.beanToMap 的 CopyOptions 配置？

```java
CopyOptions.create()
    .setIgnoreNullValue(true)                    // 忽略 null 值
    .setFieldValueEditor((k, v) -> v.toString()) // 值转 String
```

Redis Hash 的 value 都是 String，所以要强制 toString()。

---

## 八、Axios（前端）

### 8.1 Axios 是什么？

前端用来**调后端接口**的 HTTP 请求库，类似 Java 的 RestTemplate。

```javascript
axios.get('/shop/1').then(res => console.log(res.data));
axios.post('/user/login', { phone, code }).then(res => ...);
```

---

## 关联笔记

- [[hm-dianping项目总结]]
- [[登录功能完整实现]]
- [[Redis缓存三大经典问题]]
- [[缓存模块总结]]
- [[秒杀功能实现与并发安全]]
- [[一人一单与分布式锁]]
- [[Redis分布式锁实现]]
- [[Redisson可重入锁原理详解]]
