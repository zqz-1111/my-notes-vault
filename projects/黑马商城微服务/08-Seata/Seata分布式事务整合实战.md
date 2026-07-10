# 【Seata】分布式事务整合实战

## 一、写在前面

微服务拆分后，一个业务操作可能涉及多个服务的数据库操作。比如下单这个动作：

1. trade-service：创建订单（本地DB）
2. item-service：扣减库存（远程调用）
3. cart-service：清理购物车（远程调用）

`@Transactional` 只能管本地数据库。如果步骤1成功、步骤2失败，订单建了但库存没扣——**数据不一致**。

**Seata** 是阿里巴巴开源的分布式事务框架，让跨服务的多个数据库操作**要么全部成功，要么全部回滚**。

学完这篇你能掌握：
1. Seata 核心概念（TC/TM/RM）
2. Docker 部署 Seata Server
3. 微服务整合 Seata（共享配置 + @GlobalTransactional）
4. 测试分布式事务的提交与回滚

## 二、前置准备

- 已安装 Docker 的虚拟机（192.168.188.130）
- Nacos 注册中心已启动（端口 8848 + 9848）
- MySQL 已启动，已有 `seata` 数据库（含 undo_log 表）
- trade-service、item-service、cart-service 三个微服务模块

## 三、Seata 核心概念

### 3.1 三个角色

| 角色 | 全称 | 职责 | 对应项目 |
|---|---|---|---|
| **TC** | Transaction Coordinator | 事务协调者，维护全局事务状态，协调提交或回滚 | Seata Server（独立部署） |
| **TM** | Transaction Manager | 事务管理器，发起全局事务 | trade-service（发起下单） |
| **RM** | Resource Manager | 资源管理器，管理分支事务 | item-service、cart-service |

### 3.2 工作流程

```
trade-service (TM)
    │
    │ ① 开启全局事务
    ▼
Seata Server (TC)
    │
    │ ② 协调各分支事务
    ├──→ item-service (RM)："库存扣了吗？"
    └──→ cart-service (RM)："购物车清了吗？"
    
    ③ 全部成功 → 提交，任一失败 → 全部回滚
```

### 3.3 三种事务模式

| 模式 | 原理 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|---|
| **AT** | 自动生成逆向SQL回滚，靠 undo_log 表 | 代码侵入小，加注解就行 | 有全局锁，性能一般 | 大多数业务场景 |
| **TCC** | 手写 Try/Confirm/Cancel 三个方法 | 性能好，灵活 | 代码量大，业务侵入强 | 高性能要求 |
| **Saga** | 每个步骤配一个补偿操作 | 适合长事务 | 补偿逻辑复杂 | 长流程业务 |

**本项目使用 AT 模式**，最简单：加 `@GlobalTransactional` 注解 + 每个服务建 `undo_log` 表。

## 四、Docker 部署 Seata Server

### 4.1 准备配置文件

在虚拟机上创建 `~/seata/application.yml`：

```yaml
server:
  port: 7099

spring:
  application:
    name: seata-server

logging:
  config: classpath:logback-spring.xml
  file:
    path: ${user.home}/logs/seata

console:
  user:
    username: admin
    password: admin

seata:
  config:
    type: file
  registry:
    type: nacos
    nacos:
      application: seata-server
      server-addr: nacos:8848      # 用容器名，需要在同一 Docker 网络
      group: "DEFAULT_GROUP"
      namespace: ""
      username: "nacos"
      password: "nacos"
  store:
    mode: db                       # 用数据库存储事务日志
    session:
      mode: db
    lock:
      mode: db
    db:
      datasource: druid
      db-type: mysql
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://mysql:3306/seata?rewriteBatchedStatements=true&serverTimezone=UTC
      user: root
      password: 123
      min-conn: 10
      max-conn: 100
      global-table: global_table
      branch-table: branch_table
      lock-table: lock_table
      distributed-lock-table: distributed_lock
      query-limit: 1000
      max-wait: 5000
  transport:
    rpc-tc-request-timeout: 15000
```

**关键配置说明：**
- `registry.nacos.server-addr: nacos:8848` → 用容器名通信（需要 `--network hm-net`）
- `store.mode: db` → 事务日志存数据库（学习阶段可用 `file` 模式代替）
- `db.url: jdbc:mysql://mysql:3306/seata` → 用容器名访问 MySQL

### 4.2 启动容器

```bash
docker run --name seata \
  -p 8099:8099 \
  -p 7099:7099 \
  -e SEATA_IP=192.168.188.130 \
  -v ~/seata:/seata-server/resources \
  --privileged=true \
  --network hm-net \
  -d \
  seataio/seata-server:1.5.2
```

**参数说明：**
- `-p 8099:8099` → RPC 通信端口（TC 监听）
- `-p 7099:7099` → 控制台端口（Web UI）
- `-e SEATA_IP=192.168.188.130` → 注册到 Nacos 的 IP 地址
- `-v ~/seata:/seata-server/resources` → 挂载配置文件
- `--network hm-net` → 加入 Docker 网络，才能用容器名访问 nacos 和 mysql

### 4.3 验证启动

```bash
# 查看日志
docker logs --tail 5 seata

# 成功标志
# seata server started in xxxxx millSeconds
```

访问 Nacos 控制台（http://192.168.188.130:8848/nacos），服务列表应有 `seata-server`。

## 五、微服务整合 Seata

### 5.1 引入依赖（三个服务都要）

在 `trade-service`、`item-service`、`cart-service` 的 `pom.xml` 中添加：

```xml
<!--统一配置管理-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
<!--读取bootstrap文件-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
<!--seata-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

> **注意：** cart-service 之前学 Sentinel 时已经引过 nacos-config 和 bootstrap，只加 seata 就行。

### 5.2 Nacos 共享配置

在 Nacos 控制台创建共享配置 `shared-seata.yaml`（只需创建一次，所有服务共用）：

```yaml
seata:
  registry:
    type: nacos
    nacos:
      server-addr: 192.168.188.130:8848  # 你的 Nacos 地址
      namespace: ""
      group: DEFAULT_GROUP
      application: seata-server
      username: nacos
      password: nacos
  tx-service-group: hmall  # 事务组名称
  service:
    vgroup-mapping:
      hmall: "default"     # 事务组映射到 TC 集群
```

**通俗理解：** 微服务启动时，去 Nacos 找一个叫 `seata-server` 的服务，那就是 TC。

### 5.3 创建 bootstrap.yaml

每个参与分布式事务的服务都需要 `bootstrap.yaml`，在引导阶段读取 Nacos 共享配置。

以 `trade-service/src/main/resources/bootstrap.yaml` 为例：

```yaml
spring:
  application:
    name: trade-service
  profiles:
    active: dev
  cloud:
    nacos:
      server-addr: 192.168.188.130:8848
      config:
        file-extension: yaml
        shared-configs:
          - dataId: shared-jdbc.yaml
          - dataId: shared-log.yaml
          - dataId: shared-swagger.yaml
          - dataId: shared-seata.yaml  # 新增这行
```

> **关键：** 比之前只多了 `shared-seata.yaml` 这一行。其他共享配置（jdbc、log、swagger）之前就有了。

### 5.4 精简 application.yaml

通用配置已挪到 Nacos 共享配置，`application.yaml` 只保留服务独有的配置：

```yaml
# trade-service/application.yaml（精简后）
server:
  port: 8085
feign:
  okhttp:
    enabled: true
  sentinel:
    enabled: true
hm:
  swagger:
    title: 交易服务接口文档
    package: com.hmall.trade.controller
  db:
    database: hm-trade
```

**删掉的配置去向：**

| 删掉的 | 去哪了 |
|---|---|
| spring.datasource.* | shared-jdbc.yaml（Nacos） |
| mybatis-plus.* | shared-jdbc.yaml（Nacos） |
| logging.* | shared-log.yaml（Nacos） |
| knife4j.* | shared-swagger.yaml（Nacos） |
| spring.cloud.nacos.* | bootstrap.yaml（引导阶段） |

### 5.5 添加 @GlobalTransactional

在 `OrderServiceImpl.createOrder()` 方法上，把 `@Transactional` 改为 `@GlobalTransactional`：

```java
import io.seata.spring.annotation.GlobalTransactional;

@Override
@GlobalTransactional  // 替代原来的 @Transactional
public Long createOrder(OrderFormDTO orderFormDTO) {
    // 1. 创建订单（本地）
    // 2. 扣减库存（远程 - item-service）
    // 3. 清理购物车（远程 - cart-service）
}
```

**区别：**
- `@Transactional` → 只管 trade-service 自己的数据库
- `@GlobalTransactional` → 开启全局事务，TC 协调所有参与服务

## 六、踩坑避坑指南

### 报错坑

**坑 1：Seata Server 连不上 Nacos**
- 报错：`UnknownHostException: nacos`
- 原因：Seata 容器不在 `hm-net` 网络，无法解析容器名 `nacos`
- 解决：启动时加 `--network hm-net`，并确保 nacos 容器也在该网络

**坑 2：Seata Server 连不上 MySQL**
- 报错：`Connection refused: 127.0.0.1:3306`
- 原因：环境变量名写错了（`DB_HOST` 不是 Seata 1.5.2 的正确变量名）
- 解决：用配置文件方式（`-v ~/seata:/seata-server/resources`），不用环境变量

**坑 3：Seata Server 启动后立刻退出**
- 现象：`docker ps` 看不到容器
- 原因：配置有误导致启动失败，容器退出
- 解决：用 `docker logs seata` 查看具体报错

**坑 4：Nacos 环境变量不生效**
- 报错：`failed to req API:/nacos/v1/ns/instance after all servers([localhost:8848]) tried`
- 原因：Seata 1.5.2 镜像的环境变量名和我们猜的不一样
- 解决：用配置文件方式，不用环境变量

### 逻辑坑

**坑 5：Seata 配置文件里用容器名，但容器不在同一网络**
- 问题：`server-addr: nacos:8848` 解析不了
- 原因：seata 和 nacos 不在同一个 Docker 网络
- 解决：所有容器加入 `hm-net` 网络

**坑 6：ItemMapper.updateStock 找不到**
- 报错：`Mapped Statements collection does not contain value for com.hmall.item.ItemMapper.updateStock`
- 原因：`@Update` 注解 + `executeBatch` 配合有问题，MyBatis 找不到 mapped statement
- 解决：改用 `getBaseMapper().updateStock(item)` 直接调用，不用 `executeBatch`

**坑 7：OrderDetailDTO 类型不一致**
- 报错：`不兼容的类型: com.hmall.api.dto.OrderDetailDTO无法转换为com.hmall.item.domain.dto.OrderDetailDTO`
- 原因：ItemMapper 引用了 item-service 本地的 OrderDetailDTO，但服务层用的是 hm-api 共享模块的
- 解决：统一用 `com.hmall.api.dto.OrderDetailDTO`

## 七、效果验证

### 7.1 正常下单（提交）

```bash
# 登录获取 Token
curl -X POST http://localhost:8084/users/login \
  -H "Content-Type: application/json" \
  -d '{"username":"jack","password":"123"}'

# 下单
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -H "Authorization: 你的token" \
  -d '{"paymentType":1,"details":[{"itemId":14741770661,"num":1}]}'
```

**结果：**
- 返回订单 ID：`2075079276520022018`
- 库存从 10000 变成 9999
- 购物车对应商品被清除
- 全局事务提交成功

### 7.2 库存不足（回滚）

```bash
# 买库存只有1的商品，要2个
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -H "Authorization: 你的token" \
  -d '{"paymentType":1,"details":[{"itemId":2120808,"num":2}]}'
```

**结果：**
- 返回 500 错误（库存不足）
- 订单表没有新记录（回滚了）
- 库存不变（还是 1）
- Seata 日志显示：`rollback status: Rollbacked`

## 八、XA 模式 vs AT 模式

### 8.1 XA 模式原理

XA 是 X/Open 组织定义的分布式事务标准，基于**两阶段提交**：

**一阶段：**
- 事务协调者通知每个参与者执行本地事务
- 执行完成后报告状态，**不提交**，继续持有数据库锁

**二阶段：**
- 都成功 → 通知所有参与者提交
- 任一失败 → 通知所有参与者回滚

**Seata 的 XA 模型：**
1. RM 一阶段：注册分支事务 → 执行 SQL 但不提交 → 报告状态
2. TC 二阶段：检测各分支状态 → 通知提交或回滚
3. RM 二阶段：接收指令，提交或回滚

**优点：** 强一致性（ACID）、数据库原生支持、无代码侵入
**缺点：** 锁持有时间长、性能差、依赖关系型数据库

### 8.2 XA 模式实现

```java
@Configuration
public class DataSourceConfig {
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource dataSource() {
        return new DataSourceProxyXA(dataSource());  // 关键：用 XA 代理
    }
}
```

**核心区别：** AT 用 `DataSourceProxy`，XA 用 `DataSourceProxyXA`

### 8.3 AT 模式原理（回顾）

**一阶段：**
- 拦截 SQL，记录前后镜像到 `undo_log` 表
- **直接提交**，不锁资源

**二阶段提交：** 删除 undo_log
**二阶段回滚：** 用 undo_log 反向补偿（反向 SQL）

### 8.4 AT vs XA 核心区别

| 对比项 | AT 模式 | XA 模式 |
|---|---|---|
| 一阶段 | 直接提交 | 不提交 |
| 锁持有 | 短（提交即释放） | 长（等二阶段） |
| 性能 | 好 | 差 |
| 一致性 | 最终一致 | 强一致 |
| 脏读风险 | 有（一阶段已提交） | 无 |
| 代码侵入 | 需要 undo_log 表 | 无 |
| 数据库支持 | 任意 | 需支持 XA 协议 |

**选择建议：**
- 大多数场景用 **AT**（简单、性能好）
- 金融级强一致用 **XA**

---

## 九、Sentinel + Seata 冲突与解决方案

### 9.1 功能定位

- **Sentinel**：管流量（进不进来）— 熔断、降级、限流
- **Seata**：管事务（进来后怎么处理）— 分布式事务

两者功能不冲突，但组合使用有 3 个坑：

### 9.2 坑 1：超时问题

**问题：** Sentinel 熔断有超时时间，Seata 事务也需要时间。如果 Sentinel 超时 < Seata 事务时间，Sentinel 先熔断 → Seata 事务被中断。

**解决：** 配置 Seata 超时时间（在 shared-seata.yaml）：
```yaml
seata:
  client:
    tm:
      default-global-transaction-timeout: 60000  # 60秒
```

### 9.3 坑 2：异常处理

**问题：** Sentinel 的 fallback 会捕获异常，Seata 需要异常往上抛才能触发回滚。

**实际案例（本次踩坑）：**
```java
// 问题代码
try {
    itemClient.deductStock(detailDTOS);
} catch (Exception e) {
    throw new RuntimeException("库存不足！");  // 异常类型变了
}
```

`ItemClientFallback` 抛出 `BizIllegalException`，被 catch 后变成 `RuntimeException`，Seata 可能无法正确识别。

**解决：** 去掉 try-catch，让异常自然抛出：
```java
itemClient.deductStock(detailDTOS);  // 异常直接抛出，Seata 能捕获
```

### 9.4 坑 3：资源隔离

**问题：** Sentinel 线程池满时，Seata 连接拿不到 → 事务超时。

**解决：** 给 Seata 相关接口配置独立线程池（流量大时再优化）。

---

## 十、全文总结

- Seata 解决分布式事务问题，核心思想：找一个统一协调者（TC）管理多个分支事务
- 三个角色：TC（独立部署）、TM（发起事务）、RM（参与事务）
- **四种模式**：AT（默认）、TCC、Saga、XA
- **AT 模式**：一阶段直接提交 + undo_log 记录镜像，性能好但有脏读风险
- **XA 模式**：一阶段不提交，强一致但性能差
- Docker 部署 Seata Server 注意：用配置文件挂载（不用环境变量）、加入 hm-net 网络
- 微服务整合三步：加依赖 → 创建 bootstrap.yaml 引入 shared-seata.yaml → 改注解
- **Sentinel + Seata 组合注意**：超时配置、异常不要被 fallback 吞掉、资源隔离
- 代码审查发现 try-catch 会影响 Seata 回滚，去掉后异常自然抛出

## 九、pay-service 整合 Seata（余额支付场景）

### 9.1 业务场景分析

用户选择余额支付时，pay-service 需要做三件事：

```
pay-service (TM)
    │
    │ ① 开启全局事务
    ▼
    ├──→ user-service (RM)：扣减余额（远程调用）
    ├──→ pay-service (RM)：更新支付单状态（本地）
    └──→ trade-service (RM)：更新订单状态（远程调用）
    
    任意一步失败 → Seata 自动回滚所有操作
```

**风险场景：**
- 余额扣了，但支付单没更新 → 钱扣了，订单显示未支付
- 支付单更新了，但订单没更新 → 支付成功，订单状态没变

### 9.2 改造步骤

#### 第一步：pom.xml 添加依赖

```xml
<!--统一配置管理-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
<!--读取bootstrap文件-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
<!--seata-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

#### 第二步：创建 bootstrap.yaml

```yaml
spring:
  application:
    name: pay-service
  profiles:
    active: dev
  cloud:
    nacos:
      server-addr: 192.168.188.130:8848
      config:
        file-extension: yaml
        shared-configs:
          - dataId: shared-jdbc.yaml
          - dataId: shared-log.yaml
          - dataId: shared-swagger.yaml
          - dataId: shared-seata.yaml  # 关键：引入 Seata 共享配置
```

#### 第三步：精简 application.yaml

```yaml
server:
  port: 8086
feign:
  sentinel:
    enabled: true
hm:
  db:
    database: hm-pay  # 只保留服务特有配置
```

> **踩坑：** `hm.db.database` 必须配置，否则 Nacos 的 shared-jdbc.yaml 里的 `${hm.db.database}` 会解析失败。

#### 第四步：修改 PayOrderServiceImpl

```java
import io.seata.spring.annotation.GlobalTransactional;

@Override
@GlobalTransactional  // 替换原来的 @Transactional
public void tryPayOrderByBalance(PayOrderFormDTO payOrderDTO) {
    // 1.查询支付单
    // 2.判断状态
    // 3.扣减余额 ← user-service（远程调用）
    // 4.修改支付单状态 ← pay-service（本地）
    // 5.修改订单状态 ← trade-service（远程调用）
}
```

#### 第五步：创建 undo_log 表

Seata AT 模式需要在每个业务数据库创建 `undo_log` 表：

```sql
CREATE TABLE IF NOT EXISTS `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  `ext` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

> **注意：** hm-pay、hm-trade、hm-item、hm-cart 每个库都要有这张表。

### 9.3 测试验证

#### 正常支付（提交）

1. 前端登录 jack/123
2. 添加商品到购物车
3. 提交订单
4. 选择余额支付，输入正确密码 123
5. 支付成功

**结果：**
- 余额减少 ✅
- 支付单状态更新为「交易成功」 ✅
- 订单状态更新为「已支付」 ✅

#### 错误密码支付（回滚）

1. 故意输入错误密码（如 456）
2. 支付失败

**结果：**
- 余额不变 ✅
- 支付单状态还是「待支付」 ✅
- 订单状态还是「待支付」 ✅
- Seata 日志：`rollback status: Rollbacked` ✅

---

## 十、关联笔记

- [[Sentinel入门与微服务整合]]
- [[Nacos配置管理]]
- [[微服务获取用户与OpenFeign传递]]

## 十、代码变更

- 修改：`trade-service/pom.xml` — 添加 nacos-config、bootstrap、seata 依赖
- 修改：`item-service/pom.xml` — 添加 nacos-config、bootstrap、seata 依赖
- 修改：`cart-service/pom.xml` — 添加 seata 依赖（nacos-config、bootstrap 已有）
- 新增：`trade-service/src/main/resources/bootstrap.yaml` — 引入共享配置
- 新增：`item-service/src/main/resources/bootstrap.yaml` — 引入共享配置
- 修改：`cart-service/src/main/resources/bootstrap.yaml` — 添加 shared-seata.yaml
- 精简：`trade-service/src/main/resources/application.yaml` — 移除通用配置
- 精简：`item-service/src/main/resources/application.yaml` — 移除通用配置
- 修改：`trade-service/.../OrderServiceImpl.java` — @Transactional 改为 @GlobalTransactional
- 修改：`item-service/.../ItemMapper.java` — 统一使用 com.hmall.api.dto.OrderDetailDTO
- 修改：`item-service/.../ItemServiceImpl.java` — deductStock 改用 getBaseMapper().updateStock() 直接调用
