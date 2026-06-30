---
date: 2026-06-30
tags:
  - MyBatis-Plus
  - 黑马商城
  - 进阶
---

# 【MyBatis-Plus】条件构造器、自定义SQL与Service接口

## 一、写在前面

BaseMapper 只能按 ID 做简单 CRUD，实际业务需要复杂条件、自定义 SQL、批量操作。本篇讲解 MyBatis-Plus 的进阶用法。

---

## 二、条件构造器（Wrapper）

### 2.1 为什么需要 Wrapper？

```sql
-- BaseMapper 做不到的
SELECT * FROM user WHERE age > 18 AND status = 1 ORDER BY create_time DESC
UPDATE user SET balance = balance + 100 WHERE id = 1
```

### 2.2 四种 Wrapper 对比

| Wrapper | 用途 | 场景 |
|---|---|---|
| `QueryWrapper` | 查询/删除条件 | 字符串指定字段名 |
| `LambdaQueryWrapper` | 查询/删除条件 | 方法引用，编译时检查（推荐） |
| `UpdateWrapper` | 更新条件 | 复杂 set 操作 |
| `LambdaUpdateWrapper` | 更新条件 | Lambda + 复杂 set（推荐） |

### 2.3 常用条件方法

| 方法 | SQL | 示例 |
|---|---|---|
| `eq` | `= ?` | `.eq(User::getStatus, 1)` |
| `ne` | `<> ?` | `.ne(User::getStatus, 0)` |
| `gt` / `ge` | `>` / `>=` | `.gt(User::getAge, 18)` |
| `lt` / `le` | `<` / `<=` | `.lt(User::getAge, 60)` |
| `between` | `BETWEEN` | `.between(User::getAge, 18, 30)` |
| `like` | `LIKE '%?%'` | `.like(User::getUsername, "张")` |
| `likeLeft` / `likeRight` | `LIKE '%?'` / `LIKE '?%'` | `.likeRight(User::getUsername, "张")` |
| `in` | `IN (...)` | `.in(User::getId, List.of(1,2,3))` |
| `isNull` / `isNotNull` | `IS NULL` / `IS NOT NULL` | `.isNull(User::getPhone)` |
| `orderByDesc` / `orderByAsc` | `ORDER BY` | `.orderByDesc(User::getCreateTime)` |
| `select` | 指定字段 | `.select(User::getId, User::getUsername)` |

### 2.4 组合条件

```java
// AND（默认）
wrapper.eq("status", 1).gt("age", 18);
// WHERE status = 1 AND age > 18

// OR
wrapper.eq("status", 1).or().gt("age", 18);
// WHERE status = 1 OR age > 18

// 嵌套
wrapper.and(w -> w.eq("username", "张三").or().eq("phone", "13800138000"));
// WHERE (username = '张三' OR phone = '13800138000')
```

### 2.5 动态条件（实际业务常用）

```java
String username = request.getParameter("username");
Integer age = request.getParameter("age");

LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.like(StringUtils.isNotBlank(username), User::getUsername, username)
       .eq(age != null, User::getAge, age);
// 只有参数不为空才拼接条件
```

### 2.6 UpdateWrapper 用法

```java
// 普通更新（set 值固定）→ 用 QueryWrapper + entity
User user = new User();
user.setBalance(2000);
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getUsername, "jack");
userMapper.update(user, wrapper);

// 复杂更新（set 用 SQL）→ 用 LambdaUpdateWrapper
lambdaUpdate()
        .eq(User::getId, 1)
        .setSql("balance = balance + 100")
        .update();

// 更新多个字段
lambdaUpdate()
        .eq(User::getId, 1)
        .set(User::getStatus, 0)
        .set(User::getUpdateTime, LocalDateTime.now())
        .update();
```

---

## 三、自定义 SQL

### 3.1 什么时候需要自定义？

- 多表关联查询
- 复杂的子查询
- Wrapper 写不出来的 SQL

### 3.2 注解方式（@Select）

```java
@Select("SELECT * FROM user WHERE age > #{age}")
List<User> selectByAge(@Param("age") Integer age);
```

### 3.3 XML 方式

```xml
<select id="selectByCondition" resultType="User">
    SELECT * FROM user WHERE age > #{age} AND status = #{status}
</select>
```

### 3.4 自定义 SQL + Wrapper 混用（重点）

```java
@Select("SELECT * FROM user ${ew.customSqlSegment}")
List<User> selectByWrapper(@Param("ew") QueryWrapper<User> wrapper);
```

**关键**：`${ew.customSqlSegment}` 把 Wrapper 条件拼接到 SQL 中。

---

## 四、foreach 标签

### 4.1 IN 查询

```xml
<select id="selectByIds" resultType="User">
    SELECT * FROM user WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
```

### 4.2 批量插入

```xml
<insert id="batchInsert">
    INSERT INTO user (username, phone) VALUES
    <foreach collection="users" item="u" separator=",">
        (#{u.username}, #{u.phone})
    </foreach>
</insert>
```

| 属性 | 说明 |
|---|---|
| `collection` | 遍历的集合参数名 |
| `item` | 每个元素的变量名 |
| `open` | 以什么开头 |
| `close` | 以什么结尾 |
| `separator` | 分隔符 |

---

## 五、Service 接口（IService）

### 5.1 使用流程

```java
// 1. 接口继承 IService
public interface IUserService extends IService<User> {
    void deductBalance(Long userId, Integer amount);  // 自定义方法
}

// 2. 实现类继承 ServiceImpl
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements IUserService {
    // 基础 CRUD 不用写，ServiceImpl 已实现

    @Override
    public void deductBalance(Long userId, Integer amount) {
        // 业务逻辑在这里写
        User user = getById(userId);
        if (user.getStatus() != 1) throw new RuntimeException("用户状态异常");
        if (user.getBalance() < amount) throw new RuntimeException("余额不足");
        baseMapper.deductBalance(userId, amount);  // 调用自定义 SQL
    }
}
```

### 5.2 常用内置方法

| 方法 | 说明 |
|---|---|
| `save()` / `saveBatch()` | 新增 / 批量新增 |
| `removeById()` / `removeByIds()` | 删除 / 批量删除 |
| `updateById()` / `updateBatchById()` | 更新 / 批量更新 |
| `getById()` / `list()` / `listByIds()` | 单查 / 全查 / 批量查 |
| `count()` / `count(wrapper)` | 计数 |
| `page()` | 分页查询 |

### 5.3 Lambda 链式调用（推荐）

```java
// lambdaQuery() 查询
List<User> users = lambdaQuery()
        .eq(User::getStatus, 1)
        .gt(User::getAge, 18)
        .orderByDesc(User::getCreateTime)
        .list();

// lambdaUpdate() 更新
lambdaUpdate()
        .eq(User::getId, 1)
        .set(User::getStatus, 0)
        .set(User::getUpdateTime, LocalDateTime.now())
        .update();
```

**优势**：比传统 Wrapper 更简洁，类型安全，链式调用。

### 5.4 批量新增优化

```java
// 逐条插入（慢）
for (int i = 0; i < 100000; i++) {
    userService.save(buildUser(i));
}

// 批量插入（快 10 倍）
List<User> list = new ArrayList<>(1000);
for (int i = 0; i < 100000; i++) {
    list.add(buildUser(i));
    if (i % 1000 == 0) {
        userService.saveBatch(list);
        list.clear();
    }
}
```

**进一步优化**：在 JDBC URL 添加 `rewriteBatchedStatements=true`

```yaml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/mp?...&rewriteBatchedStatements=true
```

**原理**：把多条 INSERT 合并成一条，性能再提升 10 倍。

---

## 六、Controller 层标准写法

```java
@Api(tags = "用户管理接口")
@RequiredArgsConstructor
@RestController
@RequestMapping("users")
public class UserController {

    private final IUserService userService;

    @PostMapping
    @ApiOperation("新增用户")
    public void saveUser(@RequestBody UserFormDTO userFormDTO) {
        User user = BeanUtil.copyProperties(userFormDTO, User.class);
        userService.save(user);
    }

    @DeleteMapping("/{id}")
    @ApiOperation("删除用户")
    public void removeUserById(@PathVariable("id") Long userId) {
        userService.removeById(userId);
    }

    @GetMapping("/{id}")
    @ApiOperation("根据id查询用户")
    public UserVO queryUserById(@PathVariable("id") Long userId) {
        User user = userService.getById(userId);
        return BeanUtil.copyProperties(user, UserVO.class);
    }

    @GetMapping
    @ApiOperation("根据id集合查询用户")
    public List<UserVO> queryUserByIds(@RequestParam("ids") List<Long> ids) {
        List<User> users = userService.listByIds(ids);
        return BeanUtil.copyToList(users, UserVO.class);
    }
}
```

**简单 CRUD 不用写 Service 代码**，直接调用 IService 方法。

**业务逻辑接口**（如扣减余额）需要在 Service 自定义实现。

---

## 七、踩坑避坑指南

### 逻辑坑

**坑 1：并发扣减余额**
- 问题：先查询再更新有并发问题
- 解决：用 SQL 原子操作 `UPDATE user SET balance = balance - 100 WHERE id = 1`

**坑 2：saveBatch 性能**
- 问题：默认还是逐条 INSERT
- 解决：添加 `rewriteBatchedStatements=true` 参数

---

## 八、全文总结

| 层级 | 类 | 用途 |
|---|---|---|
| Mapper 层 | BaseMapper | 单表基础 CRUD |
| Mapper 层 | Wrapper | 构建查询/更新条件 |
| Service 层 | IService/ServiceImpl | 批量操作、分页、链式查询 |

**选择建议**：
- 简单 CRUD → BaseMapper / IService 内置方法
- 条件查询 → LambdaQueryWrapper / lambdaQuery()
- 复杂更新 → LambdaUpdateWrapper / lambdaUpdate()
- 批量操作 → saveBatch() + rewriteBatchedStatements
- 多表/复杂 SQL → 自定义 SQL

---

## 九、关联笔记

- [[MyBatis-Plus快速入门与常见注解]]
- [[README|黑马商城项目总览]]
