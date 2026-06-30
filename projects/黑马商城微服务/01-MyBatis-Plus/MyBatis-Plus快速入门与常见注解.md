---
date: 2026-06-30
tags:
  - MyBatis-Plus
  - 黑马商城
  - 基础
---

# 【MyBatis-Plus】快速入门与常见注解

## 一、写在前面

### 为什么学 MyBatis-Plus？

- MyBatis 单表 CRUD 要写大量重复 XML 和 SQL
- MyBatis-Plus 在 MyBatis 基础上**只做增强不做改变**，简化开发、提高效率

### 学完你能掌握什么

1. MyBatis-Plus 的基本使用流程
2. 三个核心注解的用法
3. 单表 CRUD 零 SQL 实现

---

## 二、前置准备

- Spring Boot 2.x
- MyBatis-Plus 3.5.3.1
- MySQL 数据库

---

## 三、快速入门

### 3.1 引入依赖

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.3.1</version>
</dependency>
```

> 引入后会自动装配 MyBatis 和 MyBatis-Plus，无需再引入 mybatis-spring-boot-starter。

### 3.2 定义 Mapper 接口

```java
package com.itheima.mp.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.itheima.mp.domain.po.User;

public interface UserMapper extends BaseMapper<User> {
}
```

**核心点**：继承 `BaseMapper<T>`，泛型传入实体类，即可获得单表 CRUD 能力。

### 3.3 BaseMapper 自带的方法

| 方法 | 说明 |
|---|---|
| `insert(T)` | 新增 |
| `deleteById(S)` | 根据 ID 删除 |
| `updateById(T)` | 根据 ID 更新 |
| `selectById(S)` | 根据 ID 查询 |
| `selectBatchIds(Collection)` | 批量查询 |
| `selectList(Wrapper)` | 条件查询 |

### 3.4 测试 CRUD

```java
@SpringBootTest
class UserMapperTest {

    @Autowired
    private UserMapper userMapper;

    @Test
    void testInsert() {
        User user = new User();
        user.setId(5L);
        user.setUsername("Lucy");
        user.setPassword("123");
        user.setPhone("18688990011");
        user.setBalance(200);
        user.setInfo("{\"age\": 24, \"intro\": \"英文老师\", \"gender\": \"female\"}");
        user.setCreateTime(LocalDateTime.now());
        user.setUpdateTime(LocalDateTime.now());
        userMapper.insert(user);
    }

    @Test
    void testSelectById() {
        User user = userMapper.selectById(5L);
        System.out.println("user = " + user);
    }

    @Test
    void testSelectByIds() {
        List<User> users = userMapper.selectBatchIds(List.of(1L, 2L, 3L, 4L, 5L));
        users.forEach(System.out::println);
    }

    @Test
    void testUpdateById() {
        User user = new User();
        user.setId(5L);
        user.setBalance(20000);
        userMapper.updateById(user);
    }

    @Test
    void testDelete() {
        userMapper.deleteById(5L);
    }
}
```

---

## 四、常见注解

### 4.1 @TableName

**作用**：指定实体类对应的表名（当类名和表名不一致时使用）

```java
@TableName("user")
public class User {
    private Long id;
    private String name;
}
```

### 4.2 @TableId

**作用**：指定主键字段及主键策略

```java
@TableName("user")
public class User {
    @TableId(type = IdType.AUTO)  // 数据库自增
    private Long id;
    private String name;
}
```

**主键策略**：

| 策略 | 说明 |
|---|---|
| `AUTO` | 数据库自增 |
| `INPUT` | 手动设置 id |
| `ASSIGN_ID` | 雪花算法生成 Long/String（默认） |
| `ASSIGN_UUID` | 生成 UUID 字符串 |

### 4.3 @TableField

**作用**：指定普通字段映射关系

**必须加的三种场景**：

```java
@TableName("user")
public class User {
    @TableId
    private Long id;

    // 场景1：字段名和数据库列名不一致
    @TableField("username")
    private String name;

    // 场景2：字段以 is 开头（JavaBean 会去掉 is）
    @TableField("is_married")
    private Boolean isMarried;

    // 场景3：字段名和数据库关键字冲突
    @TableField("`concat`")
    private String concat;
}
```

---

## 五、踩坑避坑指南

### 逻辑坑

**坑 1：is 开头的字段名**
- 问题：JavaBean 规范会把 `isMarried` 的 `is` 去掉，导致和数据库列名不匹配
- 解决：加 `@TableField("is_married")` 显式指定

**坑 2：数据库关键字作字段名**
- 问题：如 `order`、`desc`、`concat` 等是 MySQL 保留字
- 解决：用反引号转义 `@TableField("\`concat\`")`

---

## 六、MyBatis-Plus 使用流程总结

```
① 引入 mybatis-plus-boot-starter 依赖
② 定义 Mapper 接口继承 BaseMapper<T>
③ 实体类加注解（@TableName、@TableId、@TableField）
④ 启动类加 @MapperScan 扫描 Mapper 包
⑤ application.yml 配置数据源
```

---

## 七、关联笔记

- [[README|黑马商城项目总览]]
- [[MyBatis基础]]

## 八、代码变更

- 修改：`pom.xml` — 添加 mybatis-plus-boot-starter 依赖
- 修改：`src/main/java/com/itheima/mp/mapper/UserMapper.java` — 继承 BaseMapper
- 修改：`src/test/java/com/itheima/mp/mapper/UserMapperTest.java` — 使用 BaseMapper 方法测试
