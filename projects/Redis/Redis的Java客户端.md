---
date: 2026-06-15
tags: [Redis, 黑马程序员, Jedis, SpringDataRedis, Java]
---

# 【Redis】Java 客户端｜Jedis 和 SpringDataRedis 实战

## 一、写在前面

1. **为什么需要 Java 客户端？** 命令行操作 Redis 只是学习用，实际项目中需要用 Java 代码连接 Redis 进行读写
2. **学完这篇你能掌握：** 用 Jedis 和 SpringDataRedis 两种方式在 Java 中操作 Redis，理解各自的优缺点

## 二、前置准备

- JDK 8+
- Maven 或 Gradle 项目
- Redis 服务已启动（默认 localhost:6379）
- 前置知识：[[Redis基础入门]]、Spring Boot 基础

## 三、正文分步讲解

### 3.1 Redis Java 客户端对比

| 客户端 | 特点 | 推荐度 |
|---|---|---|
| **Jedis** | 轻量、直连、API 简单，但线程不安全 | 入门学习 |
| **Lettuce** | 基于 Netty，线程安全，Spring Boot 默认 | 生产推荐 |
| **SpringDataRedis** | Spring 封装，支持序列化，API 统一 | 项目首选 |

### 3.2 Jedis 快速入门

**引入依赖：**

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>4.4.3</version>
</dependency>
```

**基本使用：**

```java
// 1. 创建 Jedis 连接对象
Jedis jedis = new Jedis("localhost", 6379);

// 2. 设置密码（如果有）
// jedis.auth("yourpassword");

// 3. 操作 Redis
jedis.set("name", "zhangsan");
String name = jedis.get("name");
System.out.println(name);  // zhangsan

// 4. 操作 Hash
jedis.hset("user:1", "name", "lisi");
jedis.hset("user:1", "age", "25");
Map<String, String> user = jedis.hgetAll("user:1");
System.out.println(user);  // {name=lisi, age=25}

// 5. 操作 List
jedis.lpush("mylist", "a", "b", "c");
List<String> list = jedis.lrange("mylist", 0, -1);
System.out.println(list);  // [c, b, a]

// 6. 操作 Set
jedis.sadd("myset", "x", "y", "z");
Set<String> set = jedis.smembers("myset");
System.out.println(set);

// 7. 操作 SortedSet
jedis.zadd("rank", 90, "zhangsan");
jedis.zadd("rank", 85, "lisi");
Set<String> rank = jedis.zrevrange("rank", 0, -1);
System.out.println(rank);  // [zhangsan, lisi]

// 8. 关闭连接
jedis.close();
```

**Jedis 连接池（解决线程安全问题）：**

```java
// 创建连接池配置
JedisPoolConfig config = new JedisPoolConfig();
config.setMaxTotal(10);          // 最大连接数
config.setMaxIdle(5);            // 最大空闲连接
config.setMinIdle(2);            // 最小空闲连接

// 创建连接池
JedisPool pool = new JedisPool(config, "localhost", 6379);

// 从池中获取连接（用完要还回去）
try (Jedis jedis = pool.getResource()) {
    jedis.set("key", "value");
    System.out.println(jedis.get("key"));
}
// 连接自动归还到池中
```

### 3.3 SpringDataRedis 快速入门

**SpringDataRedis 是 Spring 生态的 Redis 封装**，提供了 `RedisTemplate` 工具类，统一了各种客户端的 API 差异。

**引入依赖：**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**配置 application.yml：**

```yaml
spring:
  redis:
    host: localhost
    port: 6379
    # password: yourpassword
    database: 0
```

**注入 RedisTemplate 使用：**

```java
@SpringBootTest
class RedisTest {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Test
    void testString() {
        // 设置值
        redisTemplate.opsForValue().set("name", "zhangsan");
        // 获取值
        Object name = redisTemplate.opsForValue().get("name");
        System.out.println(name);  // zhangsan
    }

    @Test
    void testHash() {
        HashOperations<String, Object, Object> hash = redisTemplate.opsForHash();
        hash.put("user:1", "name", "lisi");
        hash.put("user:1", "age", "25");
        Object name = hash.get("user:1", "name");
        System.out.println(name);  // lisi
    }

    @Test
    void testList() {
        ListOperations<String, Object> list = redisTemplate.opsForList();
        list.leftPush("mylist", "a");
        list.leftPush("mylist", "b");
        List<Object> result = list.range("mylist", 0, -1);
        System.out.println(result);  // [b, a]
    }

    @Test
    void testSet() {
        SetOperations<String, Object> set = redisTemplate.opsForSet();
        set.add("myset", "x", "y", "z");
        Set<Object> members = set.members("myset");
        System.out.println(members);
    }

    @Test
    void testSortedSet() {
        ZSetOperations<String, Object> zset = redisTemplate.opsForZSet();
        zset.add("rank", "zhangsan", 90);
        zset.add("rank", "lisi", 85);
        Set<Object> rank = zset.reverseRange("rank", 0, -1);
        System.out.println(rank);  // [zhangsan, lisi]
    }
}
```

### 3.4 RedisTemplate 序列化问题

**默认序列化的坑：** `RedisTemplate` 默认用 JDK 序列化，存到 Redis 里的数据是乱码（二进制），可读性极差。

**解决方案：配置 JSON 序列化**

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        // Key 用 String 序列化
        template.setKeySerializer(RedisSerializer.string());
        template.setHashKeySerializer(RedisSerializer.string());

        // Value 用 JSON 序列化
        template.setValueSerializer(RedisSerializer.json());
        template.setHashValueSerializer(RedisSerializer.json());

        template.afterPropertiesSet();
        return template;
    }
}
```

配置后 Redis 里存的数据就是可读的 JSON 格式了。

### 3.5 操作各种数据类型的 API 速查

| 数据类型 | 操作对象 | 获取方式 |
|---|---|---|
| String | ValueOperations | `redisTemplate.opsForValue()` |
| Hash | HashOperations | `redisTemplate.opsForHash()` |
| List | ListOperations | `redisTemplate.opsForList()` |
| Set | SetOperations | `redisTemplate.opsForSet()` |
| SortedSet | ZSetOperations | `redisTemplate.opsForZSet()` |
| 通用操作 | -- | `redisTemplate.delete()`、`redisTemplate.expire()` |

## 四、踩坑避坑指南

**坑 1：连接 Redis 报 Connection refused**
- 报错信息：`redis.clients.jedis.exceptions.JedisConnectionException: Failed connecting to host localhost:6379`
- 原因：Redis 服务没启动，或者绑定地址不是 localhost
- 解决方案：
```bash
# 检查 Redis 是否运行
redis-cli ping
# 如果没启动
redis-server
```

**坑 2：RedisTemplate 存的数据是乱码**
- 原因：默认 JDK 序列化
- 解决方案：参考上面 3.4 配置 JSON 序列化

**坑 3：Jedis 多线程下报错**
- 原因：Jedis 实例不是线程安全的
- 解决方案：使用 JedisPool 连接池，每次操作从池中获取连接

**坑 4：SpringDataRedis 连接超时**
- 报错信息：`Unable to connect to localhost:6379`
- 原因：Redis 绑定了 127.0.0.1，远程访问不了
- 解决方案：修改 `redis.conf` 中的 `bind 0.0.0.0`，并设置密码

## 五、效果验证

```java
@SpringBootTest
class RedisVerifyTest {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Test
    void verifyAll() {
        // String
        redisTemplate.opsForValue().set("test:string", "hello");
        System.out.println(redisTemplate.opsForValue().get("test:string"));

        // Hash
        redisTemplate.opsForHash().put("test:hash", "key", "value");
        System.out.println(redisTemplate.opsForHash().get("test:hash", "key"));

        // List
        redisTemplate.opsForList().leftPush("test:list", "item");
        System.out.println(redisTemplate.opsForList().range("test:list", 0, -1));

        // SortedSet
        redisTemplate.opsForZSet().add("test:zset", "player", 100);
        System.out.println(redisTemplate.opsForZSet().reverseRange("test:zset", 0, -1));

        // 清理测试数据
        redisTemplate.delete(Arrays.asList(
            "test:string", "test:hash", "test:list", "test:zset"
        ));
    }
}
```

## 六、全文总结

1. **Jedis** 是轻量级客户端，适合学习和简单场景，但需要自己管理连接池
2. **SpringDataRedis** 是项目首选，`RedisTemplate` 封装了所有操作，配合 Spring Boot 开箱即用
3. **一定要配置 JSON 序列化**，否则 Redis 里全是乱码
4. **五种数据类型对应五个 Operations 对象**，通过 `redisTemplate.opsForXxx()` 获取
5. 生产环境推荐用 **Lettuce**（Spring Boot 默认），线程安全且性能好

## 七、关联笔记

- [[Redis基础入门]]

## 八、代码变更

- 新增：`RedisConfig.java` — JSON 序列化配置
- 新增：`RedisTest.java` — 各数据类型操作测试
