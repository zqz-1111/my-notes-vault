---
date: 2026-07-02
tags:
  - MyBatis-Plus
  - 黑马商城
  - 进阶
---

# 【MyBatis-Plus】分页插件与JSON处理器

## 一、写在前面

### 为什么学这些？

- 分页查询是后端最常用的功能，手写 LIMIT 太麻烦
- 数据库存 JSON 字符串，Java 用对象，需要自动转换机制
- 分页接口返回格式需要统一，前端才好对接

### 学完你能掌握什么

1. 用 MP 分页插件实现自动分页
2. 设计通用分页返回实体（PageVO）
3. 用 JSON 处理器实现数据库 JSON 字段自动转换
4. 理解 MP 分页与 PageHelper 的区别

---

## 二、前置准备

- MyBatis-Plus 3.5.x
- Spring Boot 2.x
- 了解 [[MyBatis-Plus快速入门与常见注解|MP 基础]] 和 [[MyBatis-Plus条件构造器与Service接口|MP 进阶]]

---

## 三、分页插件

### 3.1 为什么需要分页插件？

原生 MyBatis 分页要手写 `LIMIT`，还要自己算总数：

```java
// 手动分页，麻烦
SELECT * FROM user LIMIT 0, 10
SELECT COUNT(*) FROM user  // 还要单独查总数
```

MP 分页插件自动帮你拼 SQL、查总数。

### 3.2 配置步骤

**第一步：注册分页插件（配置类）**

```java
@Configuration
public class MybatisPlusConfig {
    
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 添加分页插件，指定数据库类型
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```

**第二步：直接用**

```java
// 1. 创建分页对象（第1页，每页10条）
Page<User> page = new Page<>(1, 10);

// 2. 调用分页查询
Page<User> result = userMapper.selectPage(page, null);

// 3. 获取结果
result.getRecords();    // 当前页数据（List<User>）
result.getTotal();      // 总记录数（自动查的）
result.getPages();      // 总页数
result.getCurrent();    // 当前页码
result.getSize();       // 每页大小
```

### 3.3 Service 层用法

```java
// IService 接口提供了分页方法
Page<User> page = new Page<>(1, 10);
IPage<User> result = userService.page(page, 
    new LambdaQueryWrapper<User>()
        .eq(User::getStatus, 1)
        .orderByDesc(User::getCreateTime)
);
```

### 3.4 条件分页查询

```java
// 常见场景：带条件的分页
Page<User> page = new Page<>(pageNum, pageSize);

Page<User> result = userService.page(page, 
    new LambdaQueryWrapper<User>()
        .like(StringUtils.isNotBlank(keyword), User::getUsername, keyword)  // 模糊搜索
        .eq(status != null, User::getStatus, status)                       // 状态筛选
        .orderByDesc(User::getCreateTime)                                  // 按时间倒序
);
```

### 3.5 SQL 执行效果

```java
Page<User> page = new Page<>(2, 10);  // 第2页，每页10条
```

插件自动生成两条 SQL：

```sql
-- 第一条：查总数
SELECT COUNT(*) FROM user WHERE deleted = 0

-- 第二条：查分页数据
SELECT * FROM user WHERE deleted = 0 LIMIT 10, 10
--                                     offset  size
```

### 3.6 自定义 SQL 分页

```java
// Mapper 自定义方法也能分页，返回值用 IPage
@Select("SELECT * FROM user WHERE age > #{age}")
Page<User> selectByAge(Page<User> page, @Param("age") Integer age);
```

### 3.7 带排序的分页

```java
// 方式一：通过 Page 对象排序
Page<User> page = new Page<>(1, 10);
page.addOrder(new OrderItem("create_time", false));  // 按创建时间倒序

// 方式二：通过 Wrapper 排序
Page<User> page = new Page<>(1, 10);
userService.page(page, new LambdaQueryWrapper<User>()
        .orderByDesc(User::getCreateTime));
```

---

## 四、通用分页实体

### 4.1 为什么需要？

没有通用实体时，每个接口返回格式不一样：

```java
// 接口 A 返回
return Map.of("total", 100, "list", users);

// 接口 B 返回
return Map.of("count", 100, "data", orders);

// 前端：？？？每个接口字段名都不一样
```

有了通用实体，所有分页接口返回格式统一：

```json
{
  "total": 100,
  "pages": 10,
  "list": [...]
}
```

### 4.2 基础版实现

```java
@Data
public class PageVO<T> {
    /**
     * 总记录数
     */
    private Long total;

    /**
     * 总页数
     */
    private Long pages;

    /**
     * 当前页数据
     */
    private List<T> list;

    /**
     * 从 MP 的 Page 对象转换（无类型转换）
     */
    public static <T> PageVO<T> of(Page<T> page) {
        PageVO<T> vo = new PageVO<>();
        vo.setTotal(page.getTotal());
        vo.setPages(page.getPages());
        vo.setList(page.getRecords());
        return vo;
    }

    /**
     * 从 MP 的 Page 对象转换（带类型转换）
     */
    public static <T, R> PageVO<R> of(Page<T> page, Class<R> clazz) {
        PageVO<R> vo = new PageVO<>();
        vo.setTotal(page.getTotal());
        vo.setPages(page.getPages());
        vo.setList(BeanUtil.copyToList(page.getRecords(), clazz));
        return vo;
    }
}
```

### 4.3 使用示例

```java
// Controller 里一行搞定
@GetMapping("/user/page")
public PageVO<UserVO> getUserPage(
        @RequestParam(defaultValue = "1") Integer pageNum,
        @RequestParam(defaultValue = "10") Integer pageSize) {
    
    Page<User> page = new Page<>(pageNum, pageSize);
    Page<User> result = userService.page(page);
    
    // 返回统一格式
    return PageVO.of(result, UserVO.class);
}
```

### 4.4 前端拿到的 JSON

```json
{
  "total": 100,
  "pages": 10,
  "list": [
    {"id": 1, "username": "张三", "phone": "138****1234"},
    {"id": 2, "username": "李四", "phone": "139****5678"}
  ]
}
```

### 4.5 进阶版：带泛型转换

```java
@Data
public class PageDTO<T> {
    private Long total;
    private Long pages;
    private List<T> list;

    /**
     * 支持任意类型转换
     */
    public <R> PageDTO<R> convert(Function<T, R> converter) {
        PageDTO<R> result = new PageDTO<>();
        result.setTotal(this.total);
        result.setPages(this.pages);
        result.setList(this.list.stream()
                .map(converter)
                .collect(Collectors.toList()));
        return result;
    }
}

// 使用：更灵活
PageDTO<User> pageDTO = userService.page(page);
PageDTO<UserVO> voPage = pageDTO.convert(user -> {
    UserVO vo = BeanUtil.copyProperties(user, UserVO.class);
    vo.setPhone(user.getPhone().replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2"));
    return vo;
});
```

---

## 五、JSON 处理器

### 5.1 什么是 JSON 处理器？

数据库存 JSON 字符串，Java 用对象，类型不匹配：

```
数据库：options VARCHAR → '{"width":1920,"height":1080}'
Java：  Map<String, Object> 或自定义对象
```

传统做法要手动 JSON 序列化/反序列化，很麻烦。

### 5.2 配置步骤

**第一步：实体类字段加 `@TableField` 注解**

```java
@TableName(value = "user", autoResultMap = true)  // 必须加 autoResultMap
public class User {
    @TableId
    private Long id;
    private String username;
    
    @TableField(typeHandler = JacksonTypeHandler.class)  // JSON 处理器
    private Map<String, Object> options;  // 或自定义对象
}
```

**第二步：数据库字段用 VARCHAR 或 TEXT**

```sql
CREATE TABLE user (
    id BIGINT PRIMARY KEY,
    username VARCHAR(50),
    options VARCHAR(1000)  -- JSON 字符串
);
```

### 5.3 效果演示

```java
// 存：Map 自动转 JSON 字符串存库
Map<String, Object> options = new HashMap<>();
options.put("width", 1920);
options.put("height", 1080);
user.setOptions(options);
userMapper.insert(user);
// 数据库存入：{"width":1920,"height":1080}

// 取：JSON 字符串自动转回 Map
User user = userMapper.selectById(1L);
user.getOptions().get("width"); // 1920
```

### 5.4 支持的类型

| 类型 | 说明 |
|---|---|
| `Map<String, Object>` | 通用键值对，最灵活 |
| 自定义对象 | 比如 `UserConfig`，结构固定 |
| `List<Object>` | JSON 数组 |

### 5.5 自定义对象示例

```java
// 定义 JSON 对象
@Data
public class UserConfig {
    private Integer width;
    private Integer height;
    private String theme;
}

// 实体类使用
@TableName(value = "user", autoResultMap = true)
public class User {
    @TableField(typeHandler = JacksonTypeHandler.class)
    private UserConfig config;
}

// 使用
UserConfig config = new UserConfig();
config.setWidth(1920);
config.setHeight(1080);
config.setTheme("dark");
user.setConfig(config);
```

---

## 六、MP 分页 vs PageHelper 对比

### 6.1 PageHelper 写法

```java
// PageHelper 写法
public PageInfo<UserVO> queryUsersPage(PageQuery query) {
    // 1. 开启分页（ThreadLocal，必须在查询前调用）
    PageHelper.startPage(query.getPageNo(), query.getPageSize());
    
    // 2. 查询（直接调用 mapper，不需要包装）
    List<User> users = userMapper.selectList(null);
    
    // 3. 转换（PageHelper 会自动把 List 转成 Page 对象）
    PageInfo<User> pageInfo = new PageInfo<>(users);
    
    // 4. 返回
    return BeanUtil.copyProperties(pageInfo, PageInfo.class);
}
```

### 6.2 详细对比

| 对比项 | MP 分页插件 | PageHelper |
|---|---|---|
| **依赖** | 内置，无需额外依赖 | 需要引入 `pagehelper-spring-boot-starter` |
| **配置** | 注册一个 Bean | 配置拦截器、方言等 |
| **使用方式** | `Page` 对象 + `page()` 方法 | `PageHelper.startPage()` + `selectList()` |
| **侵入性** | 与 MP 深度集成 | 与 MyBatis 原生集成 |
| **总数查询** | 自动 | 自动 |
| **多数据库** | 需配置 `DbType` | 自动识别 |
| **排序** | 支持 `page.addOrder()` | 需手写 `ORDER BY` 或用 `PageHelper.orderBy()` |
| **ThreadLocal** | 不依赖，显式传递 | 依赖 ThreadLocal，有污染风险 |

### 6.3 MP 分页的优势

**1. 不依赖 ThreadLocal**

```java
// PageHelper 的坑：startPage 是 ThreadLocal，必须紧跟查询
PageHelper.startPage(1, 10);
// 如果中间有其他代码，可能污染分页
someOtherMethod();  // 这个方法里的查询也会被分页！
List<User> users = userMapper.selectList(null);  // 实际分页的是这个

// MP 的 Page 对象是显式传递，没有这个问题
Page<User> page = new Page<>(1, 10);
page(page, queryWrapper);  // 只有这个查询会被分页
```

**2. 与 MP 生态无缝集成**

```java
// MP 分页可以直接用 Lambda 条件构造器
page(userService.page(page, 
    new LambdaQueryWrapper<User>()
        .like(User::getUsername, keyword)
        .eq(User::getStatus, 1)
));
```

**3. 多表联查分页更清晰**

```java
// MP 分页：返回 IPage，直接用
IPage<UserVO> result = userMapper.selectUserPage(page, status);

// PageHelper：返回 List，要手动包装
PageHelper.startPage(1, 10);
List<UserVO> list = userMapper.selectUserList(status);
PageInfo<UserVO> pageInfo = new PageInfo<>(list);  // 还要再包一层
```

### 6.4 选型建议

| 场景 | 推荐 |
|---|---|
| 新项目用 MP | 用 MP 分页，统一技术栈 |
| 老项目用原生 MyBatis | 用 PageHelper，侵入小 |
| 追求简洁 | PageHelper 看起来更简洁，但 MP 简化后差不多 |
| 追求安全 | MP 分页，没有 ThreadLocal 污染风险 |

---

## 七、全文总结

| 特性 | 解决什么问题 | 关键配置 |
|---|---|---|
| 分页插件 | 自动分页，不用手写 LIMIT | `PaginationInnerInterceptor` + `Page` 对象 |
| 通用分页实体 | 统一返回格式 | `PageVO.of(page)` |
| JSON 处理器 | 数据库 JSON 字段自动转换 | `@TableField(typeHandler = JacksonTypeHandler.class)` |

---

## 八、踩坑避坑指南

### 分页插件坑

**坑 1：不注册插件，分页不生效**

```java
// 忘记注册 → 查出来是全部数据，total=0
// 必须有 MybatisPlusInterceptor 这个 Bean
```

**坑 2：多数据库要指定类型**

```java
// MySQL 用 DbType.MYSQL
// PostgreSQL 用 DbType.POSTGRE_SQL
// 根据你的数据库类型选
interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
```

### JSON 处理器坑

**坑 3：忘记加 autoResultMap**

```java
// 查询时 JSON 字段为 null
// 解决：@TableName 加 autoResultMap = true
@TableName(value = "user", autoResultMap = true)
public class User { ... }
```

**坑 4：数据库字段类型错误**

```java
// MySQL 的 JSON 类型有特殊语法，不推荐
// 建议用 VARCHAR 或 TEXT 存 JSON 字符串
```

### 通用分页实体坑

**坑 5：返回 null 列表**

```java
// page.getRecords() 可能返回 null
// 解决：用 BeanUtil.copyToList 会自动处理空列表
return PageVO.of(result, UserVO.class);  // 内部已处理 null
```

---

## 九、关联笔记

- [[MyBatis-Plus快速入门与常见注解]]
- [[MyBatis-Plus条件构造器与Service接口]]
- [[MyBatis-Plus高级特性：Db工具类、逻辑删除与枚举处理]]
- [[README|黑马商城项目总览]]
