---
date: 2026-06-15
tags: [Redis, 黑马程序员, NoSQL, 数据库]
---

# 【Redis】基础入门｜从 NoSQL 到五种核心数据结构

## 一、写在前面

1. **为什么学 Redis？** 传统关系型数据库（如 MySQL）在高并发场景下扛不住，Redis 基于内存，读写速度可达 10 万+ QPS，是解决缓存、秒杀、排行榜等问题的标配技术
2. **学完这篇你能掌握：** Redis 是什么、怎么安装、五种核心数据结构的命令操作，能独立用命令行玩转 Redis

## 二、前置准备

- 操作系统：Windows / Linux / Mac 均可
- Redis 版本：建议 6.x 或 7.x
- 前置知识：了解基本的数据库概念即可，零基础也能看

## 三、正文分步讲解

### 3.1 认识 NoSQL

**SQL（关系型数据库）** 的问题：
- 数据以表的形式存储，结构固定
- 高并发下读写性能瓶颈
- 不擅长做缓存、排行榜、消息队列等场景

**NoSQL（非关系型数据库）** 的特点：
- 不依赖固定的表结构
- 基于内存，读写极快
- 常见类型：键值存储（Redis）、文档数据库（MongoDB）、列族数据库（HBase）

**Redis 定位：** 基于内存的键值存储数据库，常用作**缓存**、**消息队列**、**分布式锁**、**排行榜**等

### 3.2 安装和运行 Redis

**Windows 安装（推荐用 WSL 或 Docker）：**

```bash
# Docker 方式最简单
docker run -d --name redis -p 6379:6379 redis:7
```

**Linux 安装：**

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install redis-server

# 启动 Redis
redis-server

# 另开终端连接
redis-cli
```

**连接测试：**

```bash
redis-cli
127.0.0.1:6379> ping
PONG   # 返回 PONG 说明连接成功
```

### 3.3 String（字符串类型）

**最常用的数据类型**，可以存字符串、数字、二进制数据。

```bash
# 设置值
SET name "zhangsan"
# 获取值
GET name            # 返回 "zhangsan"

# 设置带过期时间（秒）
SET code "1234" EX 60
# 设置带过期时间（毫秒）
SET code "1234" PX 60000

# 自增（常用于计数器、点赞数）
SET count 1
INCR count          # count = 2
INCRBY count 5      # count = 7

# 自减
DECR count          # count = 6
DECRBY count 3      # count = 3

# 同时设置多个值
MSET k1 "v1" k2 "v2" k3 "v3"
# 同时获取多个值
MGET k1 k2 k3
```

**使用场景：**
- 缓存用户信息、Token
- 短信验证码（配合 EX 设置过期时间）
- 计数器（文章阅读量、点赞数）
- 分布式锁（SETNX）

### 3.4 Hash（哈希类型）

**适合存储对象**，类似 Java 的 Map。

```bash
# 设置单个字段
HSET user:1 name "zhangsan"
HSET user:1 age 25

# 获取单个字段
HGET user:1 name    # 返回 "zhangsan"

# 设置多个字段
HSET user:1 name "zhangsan" age 25 city "beijing"

# 获取所有字段和值
HGETALL user:1

# 判断字段是否存在
HEXISTS user:1 name   # 返回 1（存在）

# 删除字段
HDEL user:1 age

# 自增字段值
HSET user:1 score 80
HINCRBY user:1 score 5   # score = 85
```

**使用场景：**
- 存储用户信息（姓名、年龄、城市等）
- 购物车（key 为用户 ID，field 为商品 ID，value 为数量）
- 对象缓存

### 3.5 List（列表类型）

**有序的字符串列表**，支持从两端插入和弹出，类似 Java 的 LinkedList。

```bash
# 从左边插入
LPUSH mylist "a" "b" "c"   # c b a

# 从右边插入
RPUSH mylist "d" "e"       # c b a d e

# 从左边弹出
LPOP mylist    # 返回 "c"

# 从右边弹出
RPOP mylist    # 返回 "e"

# 获取范围（0 到 -1 表示全部）
LRANGE mylist 0 -1

# 获取长度
LLEN mylist

# 阻塞弹出（常用于消息队列，没有元素时会等待）
BLPOP mylist 10   # 等待 10 秒
```

**使用场景：**
- 消息队列（LPUSH 生产 + BRPOP 消费）
- 最新列表（朋友圈、微博时间线）
- 栈（LPUSH + LPOP）

### 3.6 Set（集合类型）

**无序、不重复**的集合，类似 Java 的 HashSet。

```bash
# 添加元素
SADD myset "a" "b" "c"

# 获取所有元素
SMEMBERS myset

# 判断是否包含某元素
SISMEMBER myset "a"   # 返回 1（存在）

# 删除元素
SREM myset "a"

# 获取集合大小
SCARD myset

# 集合运算
SADD set1 "a" "b" "c"
SADD set2 "b" "c" "d"

# 交集（共同好友）
SINTER set1 set2      # b c

# 并集（所有好友）
SUNION set1 set2      # a b c d

# 差集（我有他没有的）
SDIFF set1 set2       # a
```

**使用场景：**
- 共同好友（交集）
- 抽奖（SRANDMEMBER 随机抽取）
- 标签系统
- 去重

### 3.7 SortedSet（有序集合类型）

**不重复、每个元素带一个分数（score）**，按分数排序。类似排行榜。

```bash
# 添加元素（带分数）
ZADD leaderboard 90 "zhangsan"
ZADD leaderboard 85 "lisi"
ZADD leaderboard 95 "wangwu"

# 获取排名（从 0 开始，升序）
ZRANK leaderboard "zhangsan"    # 返回 1

# 获取排名（降序，最常用的排行榜）
ZREVRANK leaderboard "zhangsan" # 返回 1

# 获取分数
ZSCORE leaderboard "zhangsan"   # 返回 90

# 按分数范围查询（升序）
ZRANGEBYSCORE leaderboard 80 100

# 按排名范围查询（降序，取前 3 名）
ZREVRANGE leaderboard 0 2 WITHSCORES

# 自增分数
ZINCRBY leaderboard 5 "lisi"    # lisi = 90

# 获取集合大小
ZCARD leaderboard

# 删除元素
ZREM leaderboard "lisi"
```

**使用场景：**
- 排行榜（积分、热度、销量）
- 带权重的任务队列
- 延迟队列（score 存时间戳）

## 四、踩坑避坑指南

**坑 1：Redis 中文乱码**
- 原因：redis-cli 默认不支持中文显示
- 解决方案：连接时加 `--raw` 参数
```bash
redis-cli --raw
```

**坑 2：Key 命名不规范导致冲突**
- 错误做法：直接用 `name`、`age` 这种通用名
- 正确做法：用冒号分隔的命名空间，如 `user:1:name`、`order:123:status`

**坑 3：忘记设置过期时间导致内存溢出**
- 做缓存时一定要设过期时间
- 命令：`EXPIRE key 3600`（3600 秒后过期）

## 五、效果验证

```bash
# 验证所有数据类型
redis-cli --raw

# String
SET test:string "hello"
GET test:string        # 应返回 hello

# Hash
HSET test:hash name "redis" version "7.0"
HGETALL test:hash

# List
LPUSH test:list "a" "b" "c"
LRANGE test:list 0 -1  # 应返回 c b a

# Set
SADD test:set "x" "y" "z"
SMEMBERS test:set

# SortedSet
ZADD test:zset 100 "player1" 90 "player2"
ZREVRANGE test:zset 0 -1 WITHSCORES
```

## 六、全文总结

1. **Redis 是基于内存的键值数据库**，读写极快，适合做缓存、排行榜、消息队列
2. **五种核心数据结构：** String（缓存/计数器）、Hash（对象存储）、List（队列/列表）、Set（去重/交集）、SortedSet（排行榜）
3. **Key 命名规范很重要**，用冒号分隔：`业务:对象ID:属性`
4. **缓存数据一定要设过期时间**，避免内存溢出
5. **SortedSet 是排行榜的完美方案**，分数天然支持排序

下一步：学习 Jedis 和 SpringDataRedis，用 Java 代码操作 Redis。

## 七、关联笔记

- [[Redis的Java客户端]]
