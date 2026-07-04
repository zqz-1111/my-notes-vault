---
date: 2026-07-04
tags: [interview, Spring, 反射, 依赖注入]
---

# Spring 依赖注入与反射

## 问题 1：Spring 有哪些依赖注入方式？

### 答案

| 方式 | 写法 | 特点 |
|---|---|---|
| 字段注入 | `@Autowired private Xxx xxx;` | 最常见，简单，但不好测试 |
| 构造器注入 | `@RequiredArgsConstructor` + `final` | **Spring 官方推荐**，依赖一目了然 |
| Setter 注入 | `@Autowired` 在 setter 方法上 | 很少用，适合可选依赖 |
| `@Resource` | `@Resource private Xxx xxx;` | JSR-250 标准，按名称匹配 |

### 推荐写法

```java
@Service
@RequiredArgsConstructor  // Lombok 生成包含 final 字段的构造器
public class CartServiceImpl {
    private final RestTemplate restTemplate;  // 构造器注入
}
```

**为什么推荐构造器注入？**
- `final` 保证不可变，不会被意外修改
- 依赖一目了然，看构造器就知道需要什么
- 启动时就报错（不会运行时空指针）
- 测试时直接 `new CartServiceImpl(mockRestTemplate)` 就行

### 深入追问

- `@Autowired` 和 `@Resource` 的区别？`@Autowired` 按类型匹配，`@Resource` 按名称匹配
- `@RequiredArgsConstructor` 和 `@AllArgsConstructor` 的区别？前者只包含 `final` 和 `@NonNull` 字段，后者包含所有字段
- 为什么 `@Autowired` 不推荐用在字段上？隐藏了依赖关系，不方便单元测试，无法声明为 `final`

---

## 问题 2：什么是 Java 反射？

### 答案

反射是 Java 的一种机制，**在运行时动态获取类的信息并操作它**，而不是在编译时写死。

```java
// 正常调用：编译时就确定了
User user = new User();
user.setName("张三");

// 反射调用：运行时才知道要调什么
Class<?> clazz = Class.forName("com.hmall.domain.User");
Object obj = clazz.getDeclaredConstructor().newInstance();
Method method = clazz.getMethod("setName", String.class);
method.invoke(obj, "张三");
```

### 反射能拿到什么

```
Class 对象（代表整个类）
  ├── getName()              → 类名
  ├── getFields()            → 所有 public 字段
  ├── getDeclaredFields()    → 所有字段（包括 private）
  ├── getMethods()           → 所有 public 方法
  ├── getDeclaredMethods()   → 所有方法（包括 private）
  ├── getConstructors()      → 所有构造器
  └── getAnnotations()       → 所有注解
```

### 获取 Class 对象的 3 种方式

```java
Class.forName("全类名")     → 最常用，配置文件里写类名
对象.getClass()             → 已有对象时用
类名.class                  → 编译时就知道类型
```

### 反射的代价

| | 正常调用 | 反射调用 |
|---|---|---|
| 速度 | 快 | 慢 5~10 倍 |
| 类型安全 | 编译期检查 | 运行时才报错 |
| 可读性 | 好 | 差 |

> 反射只在需要动态性的场景用（框架、配置驱动、插件机制）。业务代码能直接调就直接调。

### 深入追问

- Spring 哪里用了反射？`@Autowired` 注入、`@RequestMapping` 路由注册、MyBatis-Plus 实体字段映射、`ParameterizedTypeReference` 读泛型
- 反射为什么慢？需要运行时解析类信息，跳过了编译器优化
- `getDeclaredFields()` 和 `getFields()` 的区别？前者拿所有字段（包括 private），后者只拿 public

---

## 来源

黑马商城微服务项目 — 购物车远程调用改造时，涉及 RestTemplate 注入方式和 ParameterizedTypeReference 原理
