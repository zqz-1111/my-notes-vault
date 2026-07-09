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

## 八、全文总结

- Seata 解决分布式事务问题，核心思想：找一个统一协调者（TC）管理多个分支事务
- 三个角色：TC（独立部署）、TM（发起事务）、RM（参与事务）
- AT 模式最简单：加 `@GlobalTransactional` + 建 `undo_log` 表
- Docker 部署 Seata Server 注意：用配置文件挂载（不用环境变量）、加入 hm-net 网络
- 微服务整合三步：加依赖 → 创建 bootstrap.yaml 引入 shared-seata.yaml → 改注解
- 测试验证：正常下单提交成功，库存不足回滚成功

## 九、关联笔记

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
