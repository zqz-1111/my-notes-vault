---
date: 2026-07-08
tags: [hmall, Sentinel, SpringCloud, 微服务保护]
---

# 【Sentinel】入门与微服务整合实战

## 一、写在前面

微服务拆分后，服务之间相互调用。如果某个服务挂了或者响应很慢，调用方的线程会被阻塞，请求堆积，最终拖垮整个系统——这就是**服务雪崩**。

**Sentinel** 是阿里巴巴开源的流量治理组件，能帮我们实现：
- **限流**：控制请求速率，防止服务被压垮
- **熔断**：当下游服务异常时，快速失败，防止雪崩
- **降级**：熔断后返回兜底响应，而不是直接报错

学完这篇你能掌握：
1. Docker 部署 Sentinel Dashboard
2. 微服务整合 Sentinel
3. 理解簇点资源与懒加载机制

## 二、前置准备

- 已安装 Docker 的虚拟机/服务器
- 本地已启动 Nacos 注册中心
- cart-service 微服务模块

## 三、Docker 部署 Sentinel Dashboard

### 3.1 拉取镜像并运行

```bash
docker run -d \
  --name sentinel \
  -p 8858:8858 \
  bladex/sentinel-dashboard:1.8.6
```

运行后访问 `http://虚拟机IP:8858`，默认账号密码都是 `sentinel`。

### 3.2 镜像源踩坑

如果遇到 `net/http: request canceled while waiting for connection` 错误，说明 Docker Hub 被墙了。

**解决方案：** 修改 `/etc/docker/daemon.json`，换成可用的镜像源：

```json
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me",
    "https://docker.m.daocloud.io"
  ]
}
```

然后重启 Docker：
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

> **坑：** 网上很多镜像源（163、清华、搜狐等）到2026年已经失效了，需要找新源。

## 四、微服务整合 Sentinel

### 4.1 引入 Sentinel 依赖

在 `cart-service/pom.xml` 中添加：

```xml
<!--sentinel-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

### 4.2 配置 Dashboard 地址

在 `cart-service/application.yaml` 中添加：

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: 192.168.188.130:8858  # 改成你虚拟机的 IP
      http-method-specify: true  # 开启请求方式前缀
```

**关键配置说明：**
- `transport.dashboard`：Sentinel Dashboard 的地址
- `http-method-specify`：开启后，资源名会带上请求方式（如 `GET:/carts`、`POST:/cart/add`），方便区分不同请求方式的接口

### 4.3 Nacos gRPC 端口踩坑

**报错信息：**
```
Connection refused: no further information: /192.168.188.130:9848
Server check fail, please check server 192.168.188.130, port 9848 is available
```

**原因：** Nacos 2.x 使用 gRPC 通信，端口是主端口 + 1000（8848 + 1000 = 9848）。Docker 部署时需要额外映射这个端口。

**解决方案：** 重新运行 Nacos 容器，加上 9848 端口映射：

```bash
docker run -d \
  --name nacos \
  -p 8848:8848 \
  -p 9848:9848 \
  -p 9849:9849 \
  -e MODE=standalone \
  nacos/nacos-server:v2.1.0
```

> **坑：** 这个问题在 Gateway 整合时就遇到过，记住：**Nacos 2.x 必须映射 9848 端口**。

## 五、Sentinel 懒加载机制

**问题：** 启动 cart-service 后，Dashboard 里什么都看不到？

**原因：** Sentinel 是**懒加载**的，只有当接口被访问时，才会注册到 Dashboard。

**解决方案：** 启动服务后，先访问一次任意接口：

```bash
# 查询购物车列表
curl http://localhost:8082/cart/list -H "Authorization: 你的token"
```

或者直接用 Knife4j 浏览器访问 `http://localhost:8082/doc.html`，点任意接口调一下。

刷新 Dashboard，就能看到资源列表了。

## 六、簇点资源列表

访问接口后，Dashboard 会显示以下资源：

| 资源名 | 含义 | 说明 |
|---|---|---|
| sentinel_default_context | Sentinel 默认上下文 | 无流量记录 |
| sentinel_spring_web_context | Spring Web 上下文 | 显示总请求数 |
| GET:/carts | 业务接口 | 查询购物车列表 |
| GET:/swagger-resources | Swagger 文档请求 | 打开 Knife4j 时触发 |

**注意：** 默认情况下，Sentinel 用**路径**作为资源名，无法区分 GET/POST/DELETE 等不同请求方式。开启 `http-method-specify: true` 后，资源名变成 `GET:/carts`、`POST:/cart/add` 这样的格式，更清晰。

## 七、踩坑避坑指南

### 报错坑

**坑 1：Docker Hub 拉取超时**
- 报错信息：`Get "https://registry-1.docker.io/v2/": net/http: request canceled`
- 原因：国内网络无法访问 Docker Hub
- 解决：配置镜像加速器（参考 3.2 节）

**坑 2：Nacos gRPC 连接拒绝**
- 报错信息：`Connection refused: /192.168.188.130:9848`
- 原因：Nacos 2.x 的 gRPC 端口未映射
- 解决：Docker 运行 Nacos 时加上 `-p 9848:9848`

### 逻辑坑

**坑 3：Dashboard 看不到服务**
- 问题：启动服务后 Dashboard 资源列表为空
- 原因：Sentinel 懒加载，没有流量就不注册
- 解决：先访问一次接口，触发资源注册

## 八、效果验证

1. 启动 Sentinel Dashboard（Docker）
2. 启动 Nacos（确保 9848 端口已映射）
3. 启动 cart-service
4. 访问 `http://localhost:8082/cart/list`（带 Token）
5. 打开 Sentinel Dashboard（`http://192.168.188.130:8858`）
6. 在「簇点资源」页面看到 `GET:/carts` 等资源，说明整合成功

## 九、全文总结

- Sentinel 是微服务保护组件，解决限流、熔断、降级问题
- Docker 部署 Dashboard 很简单，注意镜像源和端口映射
- 微服务整合只需加依赖 + 配 Dashboard 地址
- Nacos 2.x 必须映射 9848 端口（gRPC 通信）
- Sentinel 懒加载，需要先访问接口才会注册资源
- 开启 `http-method-specify` 可以区分不同请求方式

## 十、关联笔记

- [[Gateway入门与路由配置]]
- [[Nacos配置管理与动态路由]]
- [[微服务获取用户全链路实现]]

## 十一、代码变更

- 修改：`cart-service/pom.xml` — 添加 Sentinel 依赖
- 修改：`cart-service/src/main/resources/application.yaml` — 添加 Sentinel Dashboard 配置和 http-method-specify 配置
