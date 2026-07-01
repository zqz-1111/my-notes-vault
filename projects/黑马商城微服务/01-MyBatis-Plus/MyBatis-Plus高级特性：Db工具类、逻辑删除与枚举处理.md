---
date: 2026-07-01
tags:
  - MyBatis-Plus
  - 黑马商城
  - 进阶
---

# 【MyBatis-Plus】Db工具类、逻辑删除与枚举处理

## 一、写在前面

### 为什么学这些？

- Service 之间互相调用会产生循环依赖，需要优雅的解决方案
- 重要数据不能物理删除，需要逻辑删除方案
- 数据库存数字，Java 用枚举，需要自动转换机制

### 学完你能掌握什么

1. 用 Db 静态工具类避免循环依赖
2. 逻辑删除的配置与使用
3. 枚举处理器实现数据库与 Java 枚举的自动转换

---

## 二、前置准备

- MyBatis-Plus 3.5.x
- Spring Boot 2.x
- 了解 [[MyBatis-Plus快速入门与常见注解|MP 基础]] 和 [[MyBatis-Plus条件构造器与Service接口|MP 进阶]]

---

## 三、静态工具类 Db

### 3.1 什么是循环依赖？

**循环依赖 = A 依赖 B，B 又依赖 A**，形成死循环。

```java
@Service
public class OrderService {
    @Autowired
    private UserService userService;  // 我需要 UserService
}

@Service
public class UserService {
    @Autowired
    private OrderService orderService;  // 我也需要 OrderService → 循环依赖！
}
```

Spring 创建过程：

```
创建 OrderService
  → 发现需要 UserService
    → 去创建 UserService
      → 发现需要 OrderService
        → OrderService 还没创建完！💥 卡住了
```

**生活类比**：你需要钥匙开门，门需要被打开才能拿到钥匙 → 谁也动不了。

Spring 默认能用三级缓存解决，但会报警告，设计上不合理。

### 3.2 Db 静态工具类

MyBatis-Plus 提供 `Db` 工具类，**不需要注入 Service**，直接用静态方法：

```java
import com.baomidou.mybatisplus.extension.toolkit.Db;

// 查询
User user = Db.getById(userId, User.class);
List<User> users = Db.listByIds(ids, User.class);
List<User> users = Db.lambdaQuery(User.class)
        .eq(User::getStatus, 1)
        .list();

// 新增
Db.save(user);
Db.saveBatch(users);

// 更新
Db.updateById(user);
Db.lambdaUpdate(User.class)
        .eq(User::getId, 1)
        .set(User::getStatus, 0)
        .update();

// 删除
Db.removeById(userId, User.class);
```

### 3.3 对比

| 方式 | 依赖注入 | 循环依赖 | 使用场景 |
|---|---|---|---|
| `@Autowired IService` | 需要 | 可能发生 | Service 内部正常调用 |
| `Db.xxx()` | 不需要 | 不会 | Service 之间互相调用 |

### 3.4 典型场景

```java
@Service
public class OrderServiceImpl extends ServiceImpl<OrderMapper, Order> {

    // 在订单服务里需要查用户信息，但不想注入 UserService
    public OrderVO getOrderDetail(Long orderId) {
        Order order = getById(orderId);
        
        // 用 Db 静态方法查用户，避免注入 UserService
        User user = Db.getById(order.getUserId(), User.class);
        
        OrderVO vo = BeanUtil.copyProperties(order, OrderVO.class);
        vo.setUsername(user.getUsername());
        return vo;
    }
}
```

### 3.5 多表查询示例

```java
@Override
public UserVO queryUserAndAddressById(Long userId) {
    // 1.查询用户
    User user = getById(userId);
    if (user == null) {
        return null;
    }
    // 2.查询收货地址（用 Db 避免注入 AddressService）
    List<Address> addresses = Db.lambdaQuery(Address.class)
            .eq(Address::getUserId, userId)
            .list();
    // 3.处理vo
    UserVO userVO = BeanUtil.copyProperties(user, UserVO.class);
    userVO.setAddresses(BeanUtil.copyToList(addresses, AddressVO.class));
    return userVO;
}
```

**本质**：Db 底层通过 ApplicationContext 动态获取 BaseMapper，不需要注入。

---

## 四、Java Stream 补充

Stream 是 Java 8 出的"集合操作工具"，在 MP 开发中经常用到。

### 4.1 对比

```java
// 传统写法
List<AddressVO> addressVOs = new ArrayList<>();
for (Address addr : addresses) {
    AddressVO vo = BeanUtil.copyProperties(addr, AddressVO.class);
    addressVOs.add(vo);
}

// Stream 写法（更简洁）
List<AddressVO> addressVOs = addresses.stream()
        .map(addr -> BeanUtil.copyProperties(addr, AddressVO.class))
        .collect(Collectors.toList());
```

### 4.2 常用方法速查

| 方法 | 作用 | 示例 |
|---|---|---|
| `filter()` | 筛选 | `.filter(u -> u.getAge() > 18)` |
| `map()` | 转换/提取 | `.map(User::getId)` |
| `collect()` | 收集成集合 | `.collect(Collectors.toList())` |
| `forEach()` | 遍历执行 | `.forEach(System.out::println)` |
| `count()` | 计数 | `.count()` |
| `sorted()` | 排序 | `.sorted(Comparator.comparing(User::getAge))` |

### 4.3 链式调用示例

```java
List<String> names = users.stream()
        .filter(u -> u.getAge() >= 18)      // 筛选成年人
        .map(User::getUsername)              // 提取用户名
        .collect(Collectors.toList());       // 收集成 List
// 结果：["张三", "李四", "王五"]
```

---

## 五、逻辑删除

### 5.1 什么是逻辑删除？

| 类型 | SQL | 数据状态 |
|---|---|---|
| 物理删除 | `DELETE FROM user WHERE id = 1` | 数据真的没了 |
| 逻辑删除 | `UPDATE user SET deleted = 1 WHERE id = 1` | 数据还在，标记为已删除 |

### 5.2 为什么需要？

| 场景 | 物理删除的后果 |
|---|---|
| 订单数据 | 删除后无法追溯，财务对账出问题 |
| 用户数据 | 删除后关联数据变成孤儿数据 |
| 误删 | 没法恢复，只能从备份还原 |
| 审计 | 监管要求保留数据，删了违法 |

### 5.3 配置步骤

**第一步：加字段**

```sql
ALTER TABLE user ADD COLUMN deleted INT DEFAULT 0 COMMENT '逻辑删除 0-未删除 1-已删除';
```

**第二步：实体类加注解**

```java
@TableName("user")
public class User {
    @TableId
    private Long id;
    private String username;
    
    @TableLogic  // 标记逻辑删除字段
    private Integer deleted;
}
```

**第三步：配置文件（可选，默认值就是 0 和 1）**

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted
      logic-delete-value: 1        # 已删除的值
      logic-not-delete-value: 0    # 未删除的值
```

### 5.4 效果演示

```java
// 调用删除方法
userMapper.deleteById(1L);
// 实际执行：UPDATE user SET deleted = 1 WHERE id = 1

// 调用查询方法
userMapper.selectById(1L);
// 实际执行：SELECT * FROM user WHERE id = 1 AND deleted = 0

// 调用条件查询
userMapper.selectList(null);
// 实际执行：SELECT * FROM user WHERE deleted = 0
```

**所有 MP 自动生成的 SQL 都会自动加上 `deleted = 0` 条件！**

### 5.5 注意事项

**坑 1：自定义 SQL 不会自动加条件**

```java
// 自定义 SQL 需要手动处理
@Select("SELECT * FROM user WHERE age > #{age} AND deleted = 0")
List<User> selectByAge(@Param("age") Integer age);
```

**坑 2：deleted 字段默认值**

- 数据库设 `DEFAULT 0`
- 新增时不用手动 set deleted，MP 会自动处理

---

## 六、枚举处理器

### 6.1 什么是枚举？

**枚举 = 一组固定的常量集合**，表示"只能选这几个值"的场景。

```java
public enum UserStatus {
    NORMAL(1, "正常"),
    FROZEN(2, "冻结"),
    CANCELLED(3, "注销");

    @EnumValue  // 标记存入数据库的字段
    private final int code;
    private final String desc;

    UserStatus(int code, String desc) {
        this.code = code;
        this.desc = desc;
    }
}
```

**好处**：
- 限定范围，不能乱写
- 代码可读，`UserStatus.NORMAL` 比 `1` 清晰
- 编译检查，写错会报错

### 6.2 为什么需要枚举处理器？

数据库存数字，Java 用枚举，类型不匹配：

```java
// 数据库：status INT → 1, 2, 3
// Java：UserStatus.NORMAL, FROZEN, CANCELLED

// 传统做法要手动转换
user.setStatus(1);  // 存数据库
if (user.getStatus() == 1) { ... }  // 查询后判断
```

### 6.3 配置步骤

**第一步：定义枚举（@EnumValue 标记存入数据库的字段）**

```java
public enum SexEnum {
    MALE(1, "男"),
    FEMALE(2, "女");

    @EnumValue
    private final int code;
    private final String desc;

    SexEnum(int code, String desc) {
        this.code = code;
        this.desc = desc;
    }
}
```

**第二步：实体类用枚举类型**

```java
@TableName(value = "user", autoResultMap = true)  // 需要 autoResultMap
public class User {
    @TableId
    private Long id;
    private String username;
    private SexEnum sex;  // 枚举类型
}
```

**第三步：配置枚举处理器**

```yaml
mybatis-plus:
  configuration:
    default-enum-type-handler: com.baomidou.mybatisplus.core.handlers.MybatisEnumTypeHandler
```

### 6.4 效果演示

```java
// 新增 —— 直接传枚举，自动存 code
User user = new User();
user.setUsername("张三");
user.setSex(SexEnum.MALE);  // 存入数据库的是 1
userMapper.insert(user);

// 查询 —— 自动转成枚举
User user = userMapper.selectById(1L);
user.getSex();           // MALE
user.getSex().getDesc(); // "男"

// 条件查询 —— 直接用枚举比较
List<User> users = lambdaQuery()
        .eq(User::getStatus, UserStatusEnum.NORMAL)  // WHERE status = 1
        .list();
```

### 6.5 注意事项

**坑 1：autoResultMap**

```java
// 如果查询时枚举不生效，加 autoResultMap = true
@TableName(value = "user", autoResultMap = true)
public class User { ... }
```

**坑 2：一个枚举只能有一个 @EnumValue**

```java
// 错误！两个字段都标记了
@EnumValue
private final int code;
@EnumValue  // 不行，只能有一个
private final String desc;
```

---

## 七、全文总结

| 特性 | 解决什么问题 | 关键配置 |
|---|---|---|
| Db 静态工具 | Service 间循环依赖 | `Db.lambdaQuery()` / `Db.getById()` |
| 逻辑删除 | 重要数据不能物理删除 | `@TableLogic` |
| 枚举处理器 | 数据库数字 ↔ Java 枚举 | `@EnumValue` + `MybatisEnumTypeHandler` |

---

## 八、踩坑避坑指南

### 逻辑坑

**坑 1：循环依赖设计问题**
- 问题：Service 间互相注入导致循环依赖
- 解决：用 Db 静态工具类替代注入

**坑 2：自定义 SQL 忘记加逻辑删除条件**
- 问题：自定义 SQL 不会自动加 `deleted = 0`
- 解决：手动在 SQL 中加条件

### 报错坑

**坑 3：枚举查询不生效**
- 报错：查询出来枚举字段为 null
- 解决：`@TableName` 加 `autoResultMap = true`

---

## 九、关联笔记

- [[MyBatis-Plus快速入门与常见注解]]
- [[MyBatis-Plus条件构造器与Service接口]]
- [[README|黑马商城项目总览]]
