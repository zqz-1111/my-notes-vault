---
date: '2026-06-22'
tags:
  - hm-dianping
  - SpringBoot
  - 注解
  - Java
---

# Spring Boot 常用注解大全（hm-dianping 项目实战整理）

## 一、写在前面

学 Spring Boot 最头疼的就是注解满天飞，每个都见过但说不清干啥的。这篇把 **hm-dianping 项目里用到的所有注解** 按功能分类整理，顺便补充项目没用到但面试常考的注解，一篇搞定。

---

## 二、Spring MVC 层注解（Controller 层）

### 2.1 `@RestController`

**作用：** 声明这是一个控制器，且所有方法返回值直接写入 HTTP 响应体（相当于 `@Controller` + `@ResponseBody` 的合体）。

```java
@RestController  // 相当于 @Controller + @ResponseBody
@RequestMapping("/shop")
public class ShopController {
    // 所有方法返回的 Result 对象会自动转成 JSON
}
```

**vs `@Controller`：** `@Controller` 用于返回页面（配合模板引擎），`@RestController` 用于返回数据（前后端分离）。

---

### 2.2 `@RequestMapping`

**作用：** 定义 URL 路径映射。可以用在类上（公共前缀）和方法上。

```java
@RestController
@RequestMapping("/shop")  // 类级别的公共前缀
public class ShopController {

    @GetMapping("/{id}")       // GET /shop/{id}
    public Result queryShopById(@PathVariable("id") Long id) { ... }

    @PostMapping               // POST /shop
    public Result saveShop(@RequestBody Shop shop) { ... }

    @PutMapping                // PUT /shop
    public Result updateShop(@RequestBody Shop shop) { ... }
}
```

**HTTP 方法简化注解：**

| 注解 | 等价于 | 用途 |
|------|--------|------|
| `@GetMapping("/path")` | `@RequestMapping(method = GET, value = "/path")` | 查询 |
| `@PostMapping("/path")` | `@RequestMapping(method = POST, value = "/path")` | 新增 |
| `@PutMapping("/path")` | `@RequestMapping(method = PUT, value = "/path")` | 修改 |
| `@DeleteMapping("/path")` | `@RequestMapping(method = DELETE, value = "/path")` | 删除 |

---

### 2.3 `@RequestParam`

**作用：** 从 URL 查询参数中取值。适合 `?key=value` 形式的参数。

```java
@GetMapping("/of/type")
public Result queryShopByType(
        @RequestParam("typeId") Integer typeId,           // 必传参数
        @RequestParam(value = "current", defaultValue = "1") Integer current  // 可选，有默认值
) { ... }
```

**常用属性：**

| 属性 | 说明 | 示例 |
|------|------|------|
| `value` / `name` | 参数名 | `@RequestParam("typeId")` |
| `required` | 是否必传，默认 true | `@RequestParam(required = false)` |
| `defaultValue` | 默认值（设了就自动 required=false） | `@RequestParam(defaultValue = "1")` |

**vs `@PathVariable`：** `@RequestParam` 取 `?key=value` 的值，`@PathVariable` 取 URL 路径中的值。

---

### 2.4 `@PathVariable`

**作用：** 从 URL 路径中提取变量。适合 RESTful 风格的 URL。

```java
@GetMapping("/{id}")
public Result queryShopById(@PathVariable("id") Long id) { ... }
// 访问 GET /shop/1 → id = 1
```

**对比：**

```
GET /shop/1              → @PathVariable("id") 取到 1
GET /shop?typeId=1       → @RequestParam("typeId") 取到 1
```

---

### 2.5 `@RequestBody`

**作用：** 从 HTTP 请求体中读取 JSON，反序列化为 Java 对象。用于 POST/PUT 提交 JSON 数据。

```java
@PostMapping
public Result saveShop(@RequestBody Shop shop) { ... }
// 请求体: {"name":"奶茶店", "typeId":1, ...} → 自动映射到 Shop 对象
```

**注意：** 不加 `@RequestBody` 的话，Spring 会按表单参数处理（`key=value`），不是 JSON。

---

### 2.6 `@ResponseBody`

**作用：** 方法返回值直接写入响应体，而不是跳转页面。

```java
@ResponseBody  // 返回 JSON，不是页面跳转
public Result saveShop(@RequestBody Shop shop) { ... }
```

**实际开发中很少单独用**，因为 `@RestController` 已经包含了它。

---

### 2.7 `@CrossOrigin`

**作用：** 解决跨域问题。允许其他域名的前端访问这个接口。

```java
@CrossOrigin(origins = "http://localhost:8080")  // 允许 8080 端口的前端访问
@RestController
public class ShopController { ... }
```

**项目没用到**，因为 Nginx 做了反向代理，前后端同源。

---

## 三、Spring 依赖注入注解

### 3.1 `@Resource`（项目主力）

**作用：** 按名称注入 Bean（JSR-250 标准，`javax.annotation` 包）。

```java
@Resource
public IShopService shopService;

@Resource
private StringRedisTemplate stringRedisTemplate;
```

**注入顺序：**
1. 先按属性名找 Bean（比如 `shopService` → 找名字叫 `shopService` 的 Bean）
2. 找不到再按类型找

**vs `@Autowired`：** 详见 [[#3.2 @Autowired]] 对比表。

---

### 3.2 `@Autowired`

**作用：** 按类型注入 Bean（Spring 自己的注解）。

```java
@Autowired
private IShopService shopService;
```

**注入顺序：**
1. 按类型找（`IShopService` 类型的 Bean）
2. 找到多个再按属性名匹配

**项目中只在 `ShopTypeServiceImpl` 用了一次**，其他地方全用的 `@Resource`。

---

### `@Resource` vs `@Autowired` 对比

| 对比项 | `@Resource` | `@Autowired` |
|--------|-------------|--------------|
| 来源 | JSR-250（Java 标准） | Spring 框架 |
| 默认注入方式 | 先按名称，再按类型 | 先按类型，再按名称 |
| 能否指定名称 | `@Resource(name="xxx")` | 需要配合 `@Qualifier` |
| 推荐场景 | 接口有多个实现时 | 接口只有一个实现时 |

**面试话术：** "项目用 `@Resource` 因为它是 Java 标准，不强耦合 Spring，而且按名称注入更精确。"

---

### 3.3 `@Qualifier`

**作用：** 当同一类型有多个 Bean 时，指定用哪个。必须配合 `@Autowired` 使用。

```java
@Autowired
@Qualifier("redissonClient")  // 指定注入名为 redissonClient 的 Bean
private RedissonClient client;
```

**项目没用到**，因为每个类型只有一个实现。

---

### 3.4 `@Value`

**作用：** 从配置文件中取值注入到字段。

```java
@Value("${server.port}")     // 从 application.yml 取
private String port;

@Value("${spring.redis.host:localhost}")  // 有默认值
private String redisHost;
```

**项目没用到**，用的是 `StringRedisTemplate` 直接连接。

---

## 四、Spring Bean 生命周期注解

### 4.1 `@Component`

**作用：** 把类注册为 Spring Bean，通用注解。

```java
@Component
public class CacheClient { ... }  // Spring 自动创建并管理这个对象
```

**派生注解（功能一样，语义更明确）：**

| 注解 | 用于 |
|------|------|
| `@Component` | 通用组件 |
| `@Service` | 业务逻辑层 |
| `@Repository` | 数据访问层 |
| `@Controller` | 控制器层 |

---

### 4.2 `@Service`

**作用：** 声明这是业务逻辑层的 Bean。项目里所有 ServiceImpl 都用这个。

```java
@Service
public class ShopServiceImpl extends ServiceImpl<ShopMapper, Shop> implements IShopService { ... }
```

---

### 4.3 `@Configuration`

**作用：** 声明这是一个配置类，里面可以定义 `@Bean` 方法。

```java
@Configuration
public class RedissonConfig {
    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://127.0.0.1:6379");
        return Redisson.create(config);
    }
}
```

---

### 4.4 `@Bean`

**作用：** 在 `@Configuration` 类中定义一个 Bean，Spring 会管理它的生命周期。

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
    return interceptor;
}
```

**vs `@Component`：** `@Component` 标在类上，Spring 自动扫描；`@Bean` 标在方法上，手动控制创建逻辑（比如要传参数、做配置）。

---

### 4.5 `@PostConstruct`

**作用：** Bean 初始化完成后自动调用（在构造方法之后、`@Bean` 返回之后）。

```java
@PostConstruct
public void init() {
    // 项目启动时执行一次
    BELOCK_EXECUTOR.submit(this::handleSeckillOrder);
}
```

**项目用在 `VoucherOrderServiceImpl`**，启动时开启异步线程消费秒杀订单。

**执行顺序：** 构造方法 → `@PostConstruct` → 使用 → `@PreDestroy`

---

## 五、MyBatis-Plus 注解

### 5.1 `@TableName`

**作用：** 指定实体类对应的数据库表名。

```java
@Data
@TableName("tb_shop")  // 对应数据库的 tb_shop 表
public class Shop implements Serializable {
    ...
}
```

**不加会怎样：** 默认按类名去找表（比如 `Shop` → `shop`），名字对不上就报错。

---

### 5.2 `@TableId`

**作用：** 指定主键字段和主键生成策略。

```java
@TableId(value = "id", type = IdType.AUTO)  // 字段名 id，自增策略
private Long id;
```

**主键策略 `IdType`：**

| 策略 | 说明 | 示例 |
|------|------|------|
| `AUTO` | 数据库自增 | `Shop.id`、`User.id` |
| `INPUT` | 手动输入 | `VoucherOrder.id`（用 RedisIdWorker 生成） |
| `ASSIGN_ID` | 雪花算法（Long/String） | — |
| `ASSIGN_UUID` | UUID 字符串 | — |

**项目实际用法：** 大部分表用 `AUTO`，`VoucherOrder` 和 `SeckillVoucher` 用 `INPUT`（因为 ID 由 Redis 生成）。

---

### 5.3 `@TableField`

**作用：** 字段映射配置。

```java
@TableField(exist = false)  // 这个字段在数据库表中不存在
private Double distance;

@TableField("user_name")    // 字段名和数据库列名不一致时手动映射
private String userName;
```

**常用属性：**

| 属性 | 说明 | 示例 |
|------|------|------|
| `exist = false` | 非数据库字段 | `Shop.distance`、`Voucher.seckillVoucher` |
| `value` | 指定列名 | `@TableField("user_name")` |
| `select = false` | 查询时不返回这个字段 | 密码字段 |

**项目里 `exist = false` 用得最多：** `Shop.distance`、`Voucher.seckillVoucher`、`Blog.liked`、`Blog.comments`、`ShopType.typeName` 等。

---

## 六、Lombok 注解

### 6.1 `@Data`

**作用：** 自动生成 getter、setter、toString、equals、hashCode 方法。

```java
@Data
public class Shop {
    private Long id;
    private String name;
    // 不用手写 getId()、setName()、toString() 了
}
```

**项目所有实体类和 DTO 都用了。**

---

### 6.2 `@Slf4j`

**作用：** 自动生成日志对象 `log`，不用手动写 `private static final Logger log = LoggerFactory.getLogger(...)`。

```java
@Slf4j
@RestController
public class UserController {
    public void someMethod() {
        log.info("用户登录：{}", userId);  // 直接用 log
    }
}
```

**项目用在：** `UserController`、`UploadController`、`WebExceptionAdvice`、`CacheClient`、`VoucherOrderServiceImpl`。

---

### 6.3 `@NoArgsConstructor` / `@AllArgsConstructor`

**作用：** 生成无参构造方法 / 全参构造方法。

```java
@Data
@NoArgsConstructor      // 无参构造
@AllArgsConstructor     // 全参构造
public class Result {
    private Boolean success;
    private String errorMsg;
    private Object data;
    private Long total;
}
```

**为什么要无参构造：** JSON 反序列化（Jackson）默认需要无参构造方法，没它会报错。

---

### 6.4 `@Accessors(chain = true)`

**作用：** 开启链式调用，setter 返回 `this` 而不是 `void`。

```java
@Accessors(chain = true)
public class Shop {
    private String name;
    private Long typeId;
}

// 链式调用
shop.setName("奶茶店").setTypeId(1L);
// 不用链式的话得这样：
shop.setName("奶茶店");
shop.setTypeId(1L);
```

**项目所有实体类都用了。**

---

### 6.5 `@EqualsAndHashCode(callSuper = false)`

**作用：** 生成 equals 和 hashCode 方法时**不包含父类属性**。

```java
@Data
@EqualsAndHashCode(callSuper = false)  // 只比较本类属性，忽略父类
public class Shop extends BaseEntity { ... }
```

**为什么要 `callSuper = false`：** MyBatis-Plus 的实体类通常继承了框架的基类，如果把父类属性也算进 equals/hashCode，会导致比较逻辑不符合预期。

---

## 七、Spring Boot 核心注解

### 7.1 `@SpringBootApplication`

**作用：** 启动类注解，相当于 `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`。

```java
@SpringBootApplication
@MapperScan("com.hmdp.mapper")  // 扫描 Mapper 接口
@EnableAspectJAutoProxy(exposeProxy = true)  // 开启 AOP 代理
public class HmDianPingApplication {
    public static void main(String[] args) {
        SpringApplication.run(HmDianPingApplication.class, args);
    }
}
```

**自动装配原理（面试高频）：**
1. `@SpringBootApplication` 包含 `@EnableAutoConfiguration`
2. `@EnableAutoConfiguration` 会读 `META-INF/spring.factories` 文件
3. 根据 classpath 里的依赖自动配置 Bean（比如有 Redis 依赖就自动配 `StringRedisTemplate`）

---

### 7.2 `@MapperScan`

**作用：** 指定 MyBatis Mapper 接口的扫描路径。

```java
@MapperScan("com.hmdp.mapper")  // 扫描 com.hmdp.mapper 下所有接口
```

**不加会怎样：** Spring 找不到 Mapper 接口，注入时报 `NoSuchBeanDefinitionException`。

---

### 7.3 `@EnableAspectJAutoProxy(exposeProxy = true)`

**作用：** 开启 AOP 代理，并把代理对象暴露到 `AopContext` 中。

```java
@EnableAspectJAutoProxy(exposeProxy = true)
```

**项目用在：** 秒杀功能中通过 `AopContext.currentProxy()` 获取代理对象，解决 `@Transactional` 在同类方法调用时失效的问题。

---

## 八、事务与切面注解

### 8.1 `@Transactional`

**作用：** 声明式事务管理。方法内所有数据库操作要么全成功，要么全回滚。

```java
@Transactional
public Result createVoucherOrder(Long voucherId) {
    // 1. 扣库存
    // 2. 创建订单
    // 任何一步异常都会回滚
}
```

**注意事项（踩坑点）：**
- 只能用在 `public` 方法上
- 同类内部调用会失效（因为代理机制）→ 用 `AopContext.currentProxy()` 解决
- 默认只对 `RuntimeException` 回滚，要对所有异常回滚需加 `@Transactional(rollbackFor = Exception.class)`

---

### 8.2 `@ExceptionHandler`

**作用：** 全局异常处理，捕获指定类型的异常并返回统一响应。

```java
@RestControllerAdvice  // 声明这是全局异常处理器
@Slf4j
public class WebExceptionAdvice {
    @ExceptionHandler(RuntimeException.class)  // 捕获所有运行时异常
    public Result handleRuntimeException(RuntimeException e) {
        log.error("运行时异常：", e);
        return Result.fail("服务器错误");
    }
}
```

**`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`**，是 Spring MVC 的全局增强注解。

---

## 九、序列化相关注解

### 9.1 `@JsonIgnore`

**作用：** JSON 序列化时忽略这个字段（不返回给前端）。

```java
@JsonIgnore
private LocalDateTime updateTime;  // JSON 响应中不会出现 updateTime
```

**项目用在 `ShopType`** 的 `updateTime` 和 `createTime` 字段。

---

## 十、项目未用到但面试常考的注解

### 10.1 `@Scope`

**作用：** 指定 Bean 的作用域。

```java
@Component
@Scope("prototype")  // 每次注入都创建新实例（默认是 singleton）
public class MyBean { ... }
```

| 作用域 | 说明 |
|--------|------|
| `singleton` | 默认，整个容器一个实例 |
| `prototype` | 每次请求创建新实例 |
| `request` | 每次 HTTP 请求一个实例 |
| `session` | 每个会话一个实例 |

---

### 10.2 `@ConditionalOnProperty`

**作用：** 根据配置文件的值决定是否创建这个 Bean。

```java
@Bean
@ConditionalOnProperty(name = "cache.type", havingValue = "redis")
public CacheManager cacheManager() {
    return new RedisCacheManager();
}
```

---

### 10.3 `@Async`

**作用：** 异步执行方法（在另一个线程中运行）。

```java
@Async
public void sendEmail(String to) {
    // 异步发送邮件，不阻塞主线程
}
```

**需要配合 `@EnableAsync` 使用。**

---

### 10.4 `@Scheduled`

**作用：** 定时任务。

```java
@Scheduled(cron = "0 0 2 * * ?")  // 每天凌晨 2 点执行
public void cleanExpiredData() {
    // 清理过期数据
}
```

**需要配合 `@EnableScheduling` 使用。**

---

## 十一、注解速查表

| 注解 | 所属层 | 一句话说明 |
|------|--------|-----------|
| `@RestController` | Controller | 声明 REST 控制器 |
| `@RequestMapping` | Controller | URL 路径映射 |
| `@GetMapping` | Controller | GET 请求映射 |
| `@PostMapping` | Controller | POST 请求映射 |
| `@PutMapping` | Controller | PUT 请求映射 |
| `@DeleteMapping` | Controller | DELETE 请求映射 |
| `@RequestParam` | Controller | 从 ?key=value 取参数 |
| `@PathVariable` | Controller | 从 URL 路径取参数 |
| `@RequestBody` | Controller | 从请求体读 JSON |
| `@Resource` | 通用 | 按名称注入 Bean |
| `@Autowired` | 通用 | 按类型注入 Bean |
| `@Component` | 通用 | 注册为 Bean |
| `@Service` | Service | 声明业务层 Bean |
| `@Configuration` | Config | 声明配置类 |
| `@Bean` | Config | 手动定义 Bean |
| `@PostConstruct` | 通用 | 初始化后执行 |
| `@Transactional` | Service | 声明式事务 |
| `@TableName` | Entity | 映射数据库表名 |
| `@TableId` | Entity | 主键策略 |
| `@TableField` | Entity | 字段映射 |
| `@Data` | Entity/DTO | 生成 getter/setter |
| `@Slf4j` | 通用 | 自动生成 log 对象 |
| `@JsonIgnore` | Entity | JSON 忽略字段 |
| `@SpringBootApplication` | 启动类 | 自动装配 |
| `@MapperScan` | 启动类 | 扫描 Mapper |

---

## 十二、关联笔记

- [[SpringBoot整合MyBatis]]
- [[登录功能实现]]
- [[Redis缓存三大经典问题]]
- [[分布式锁原理详解]]

## 十三、代码变更

- 无（纯知识整理笔记）
