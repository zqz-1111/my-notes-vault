---
date: 2026-07-02
tags: [Docker, 黑马商城, 基础]
---

# 【Docker】快速入门与数据卷

## 一、写在前面

### 为什么学 Docker？

- 开发环境"在我电脑上能跑"，换台机器就挂 → Docker 让环境一致
- 部署 MySQL、Redis 等软件不用手动装，一条命令搞定
- 微服务项目几十个服务，用 Docker 管理方便

### 学完你能掌握什么

1. 用 Docker 快速部署 MySQL
2. 理解 docker run 命令每个参数的含义
3. 掌握镜像、容器、数据卷的常见命令
4. 理解数据卷的作用和两种用法

---

## 二、快速入门：部署 MySQL

### 2.1 一条命令部署 MySQL

```bash
docker run -d \
  --name mysql \
  -p 3306:3306 \
  -e TZ=Asia/Shanghai \
  -e MYSQL_ROOT_PASSWORD=123 \
  mysql
```

### 2.2 逐段解读

| 命令/参数 | 作用说明 |
|---|---|
| `docker run` | 创建并启动一个新容器 |
| `-d` | 后台运行容器（detach 模式），不占用当前终端 |
| `--name mysql` | 给容器指定一个唯一名称 mysql，方便后续管理 |
| `-p 3306:3306` | 端口映射：宿主机端口:容器端口，外部可通过宿主机访问 MySQL |
| `-e TZ=Asia/Shanghai` | 设置环境变量，指定容器时区为上海时间 |
| `-e MYSQL_ROOT_PASSWORD=123` | 设置环境变量，初始化 MySQL 的 root 用户密码为 123 |
| `mysql` | 指定使用的镜像名称（默认拉取 Docker Hub 上的官方 mysql 镜像） |

### 2.3 关键点

**端口映射 `-p`：**
- 前面是宿主机端口，后面是容器内部端口
- 两者可以不同，比如 `-p 3307:3306` 就用宿主机 3307 端口访问
- 连接 MySQL 时：`jdbc:mysql://宿主机IP:映射的端口/数据库名`

**环境变量 `-e`：**
- 很多镜像初始化配置的通用方式
- MySQL、Redis、Nginx 等都支持通过 `-e` 传入配置参数

**两个 `mysql` 的区别：**
- `--name mysql` → 容器名叫 mysql（可以改成别的，比如 `--name mydb`）
- 最后的 `mysql` → 镜像名（用哪个镜像来创建容器）
- 两者完全独立，改哪个都行

---

## 三、Docker 常见命令

### 3.1 镜像相关

| 命令 | 作用 | 示例 |
|---|---|---|
| `docker pull` | 拉取镜像 | `docker pull mysql:8.0` |
| `docker images` | 查看本地所有镜像 | `docker images` |
| `docker rmi` | 删除镜像 | `docker rmi mysql:8.0` |
| `docker build` | 用 Dockerfile 构建镜像 | `docker build -t myapp:1.0 .` |
| `docker push` | 推送镜像到仓库 | `docker push myapp:1.0` |
| `docker save` | 导出镜像为 tar 文件 | `docker save -o mysql.tar mysql:8.0` |
| `docker load` | 从 tar 文件导入镜像 | `docker load -i mysql.tar` |

### 3.2 容器相关（最常用）

| 命令               | 作用               | 示例                                              |
| ---------------- | ---------------- | ----------------------------------------------- |
| `docker run`     | 创建并启动容器          | `docker run -d --name mysql -p 3306:3306 mysql` |
| `docker ps`      | 查看**运行中**的容器     | `docker ps`                                     |
| `docker ps -a`   | 查看**所有**容器（含停止的） | `docker ps -a`                                  |
| `docker stop`    | 停止容器             | `docker stop mysql`                             |
| `docker start`   | 启动已停止的容器         | `docker start mysql`                            |
| `docker restart` | 重启容器             | `docker restart mysql`                          |
| `docker rm`      | 删除容器（要先停止）       | `docker rm mysql`                               |
| `docker logs`    | 查看容器日志           | `docker logs -f mysql`（`-f` 持续跟踪）               |
| `docker exec`    | 进入容器内部执行命令       | `docker exec -it mysql bash`                    |

**最高频的 5 个命令：**

```bash
# 1. 跑起来一个容器
docker run -d --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123 mysql

# 2. 看看哪些容器在跑
docker ps

# 3. 看日志排查问题
docker logs -f mysql

# 4. 进容器里面操作
docker exec -it mysql bash

# 5. 停掉再删掉
docker stop mysql && docker rm mysql
```

---

## 四、数据卷

### 4.1 什么是数据卷？

**问题：** 容器删了，里面的数据就没了。

```bash
# 跑了个 MySQL，建了库、插了数据
docker run -d --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123 mysql

# 手滑删了容器
docker rm -f mysql

# 😭 数据全没了，因为数据存在容器内部，容器删了数据跟着没了
```

**数据卷就是解决这个问题的。** 它是一个**独立于容器的存储空间**，容器删了，数据卷里的数据还在。

**类比：**
- 容器 = 租的房子
- 容器内数据 = 房子里的家具
- 数据卷 = 仓库（独立空间）
- 挂载数据卷 = 把家具放仓库，而不是放房子里

房子（容器）拆了，仓库（数据卷）还在，家具（数据）不会丢。

### 4.2 数据卷命令

| 命令 | 作用 | 示例 |
|---|---|---|
| `docker volume create` | 创建数据卷 | `docker volume create mysql-data` |
| `docker volume ls` | 查看所有数据卷 | `docker volume ls` |
| `docker volume rm` | 删除数据卷 | `docker volume rm mysql-data` |
| `docker volume inspect` | 查看数据卷详情（存在哪） | `docker volume inspect mysql-data` |

**实际使用时不用手动创建**，跑容器时 `-v` 指定，没有会自动创建。

### 4.3 两种用法

**用法一：数据卷挂载**

```bash
docker run -d --name mysql -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=123 \
  -v mysql-data:/var/lib/mysql \
  mysql
```

- `mysql-data` → 数据卷的名字（没有会自动创建）
- `/var/lib/mysql` → 容器内 MySQL 存数据的目录
- 效果：MySQL 的数据实际存在数据卷 `mysql-data` 里，不在容器里
- 删容器？数据还在数据卷里，换个新容器挂载同一个卷，数据就回来了

**用法二：挂载本地目录**

```bash
docker run -d --name mysql -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=123 \
  -v /my/host/mysql:/var/lib/mysql \
  mysql
```

- `/my/host/mysql` → 宿主机上的目录（你电脑上的真实路径）
- `/var/lib/mysql` → 容器内的目录
- 效果：数据直接存在你电脑上，你随时能看到、能备份

### 4.4 两种用法对比

| | 数据卷挂载 | 本地目录挂载 |
|---|---|---|
| 数据存在哪 | Docker 管理的隐藏目录 | 你指定的宿主机目录 |
| 你能直接看到吗 | 不能，要用 `docker volume inspect` | 能，就在你电脑上 |
| 适合场景 | 正式部署，不需要手动改数据 | 开发调试，需要直接查看/改文件 |

---

## 五、Dockerfile

### 5.1 镜像结构

镜像不是一个文件，是**一层一层叠起来的**：

```
┌─────────────────────────┐
│   你的应用（第4层）        │  ← 比如你的 Java jar 包
├─────────────────────────┤
│   项目依赖（第3层）        │  ← 比如 JDK
├─────────────────────────┤
│   基础环境（第2层）        │  ← 比如 Ubuntu、CentOS
├─────────────────────────┤
│   基础镜像（第1层）        │  ← 比如空镜像 scratch
└─────────────────────────┘
```

**为什么要分层？** 复用。比如你有 10 个 Java 项目，JDK 那层是一样的，不用重复存 10 份。

### 5.2 Dockerfile 是啥？

Dockerfile 就是一个**文本文件**，里面写的是"怎么一层一层构建镜像"的指令。

类比：Dockerfile = 菜谱 → 镜像 = 做好的菜 → 容器 = 吃这道菜

### 5.3 常用指令

| 指令 | 作用 | 示例 |
|---|---|---|
| `FROM` | 基于哪个基础镜像（必写，第一行） | `FROM openjdk:17` |
| `ENV` | 设置环境变量 | `ENV JAVA_HOME=/usr/local/jdk` |
| `COPY` | 把宿主机文件复制到镜像里 | `COPY app.jar /app.jar` |
| `RUN` | 在镜像构建时执行命令 | `RUN apt-get install -y vim` |
| `EXPOSE` | 声明容器运行时的端口（文档作用） | `EXPOSE 8080` |
| `WORKDIR` | 设置工作目录 | `WORKDIR /app` |
| `ENTRYPOINT` | 容器启动时执行的命令 | `ENTRYPOINT ["java", "-jar", "/app.jar"]` |

**RUN vs ENTRYPOINT 区别：**

| | RUN | ENTRYPOINT |
|---|---|---|
| 什么时候执行 | 构建镜像时 | 启动容器时 |
| 用途 | 安装软件、编译代码 | 定义容器启动命令 |
| 执行几次 | 构建时执行一次 | 每次启动容器都执行 |

### 5.4 两种 Java 项目 Dockerfile 写法

**写法一：基于 Ubuntu 手动安装 JDK**

```dockerfile
# 指定基础镜像
FROM ubuntu:16.04

# 配置环境变量，JDK的安装目录、容器内时区
ENV JAVA_DIR=/usr/local

# 拷贝jdk和java项目的包
COPY ./jdk8.tar.gz $JAVA_DIR/
COPY ./docker-demo.jar /tmp/app.jar

# 安装JDK
RUN cd $JAVA_DIR \
 && tar -xf ./jdk8.tar.gz \
 && mv ./jdk1.8.0_144 ./java8

# 配置环境变量
ENV JAVA_HOME=$JAVA_DIR/java8
ENV PATH=$PATH:$JAVA_HOME/bin

# 入口，java项目的启动命令
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

**写法二：直接基于 OpenJDK 镜像（推荐）**

```dockerfile
# 基础镜像
FROM openjdk:11.0-jre-buster

# 拷贝jar包
COPY docker-demo.jar /app.jar

# 入口
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### 5.5 两种写法对比

| 对比项 | Ubuntu + 手动装 JDK | 直接用 OpenJDK 镜像 |
|---|---|---|
| 基础镜像 | ubuntu:16.04（通用 Linux 系统） | openjdk:11.0-jre-buster（预装 JRE） |
| JDK 安装 | 需手动拷贝、解压、配置环境变量 | 镜像已内置，无需额外操作 |
| 构建步骤 | 步骤多（解压、移动、配置），易出错 | 步骤极少，仅需拷贝 jar 包 |
| 镜像体积 | 大（含完整 Ubuntu 系统 + JDK） | 小（仅含 JRE 和运行环境） |
| 维护成本 | 高（需手动管理 JDK 版本、补丁） | 低（官方镜像维护） |
| 适用场景 | 需自定义系统环境、安装其他依赖 | 纯 Java 应用部署（绝大多数场景） |

### 5.6 构建与运行

```bash
# 构建镜像（在 Dockerfile 所在目录执行）
# -t 给镜像起名:版本号  . 表示当前目录
docker build -t myapp:1.0 .

# 运行容器
docker run -d --name myapp -p 8080:8080 myapp:1.0
```

### 5.7 最佳实践

1. **优先使用官方 JDK 镜像**：构建快、体积小、维护简单
2. **版本明确**：生产环境指定具体版本（如 `openjdk:11-jre-slim`），避免用 `latest`
3. **简化镜像结构**：没有额外系统依赖就别基于 Ubuntu，减少复杂度和安全风险

---

## 六、容器网络互联

### 6.1 问题：容器之间怎么互相访问？

假设部署了 Java 项目 + MySQL，分两个容器跑：

```bash
docker run -d --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123 mysql
docker run -d --name myapp -p 8080:8080 myapp:1.0
```

Java 项目里 JDBC 连接串写 `localhost` 连不上，因为 `localhost` 指的是容器自己内部，不是宿主机。

### 6.2 解法：Docker 网络

**同一个网络里的容器，可以用容器名互相访问。**

```bash
# 1. 创建网络
docker network create hmall

# 2. 容器加入网络
docker run -d --name mysql --network hmall -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123 mysql
docker run -d --name hmall --network hmall -p 8080:8080 hmall:1.0
```

配置里用容器名当域名：

```yaml
url: jdbc:mysql://mysql:3306/taobao  # ✅ 容器名当域名，能连上
```

### 6.3 Docker 网络类型

| 网络类型 | 作用 | 使用场景 |
|---|---|---|
| `bridge` | 默认网络，容器之间隔离 | 单机多容器通信（最常用） |
| `host` | 容器直接用宿主机网络 | 需要高性能网络 |
| `none` | 无网络 | 完全隔离的容器 |

99% 的情况用 bridge（默认），手动创建的网络就是 bridge 类型。

### 6.4 网络相关命令

| 命令 | 作用 | 示例 |
|---|---|---|
| `docker network create` | 创建网络 | `docker network create mynet` |
| `docker network ls` | 查看所有网络 | `docker network ls` |
| `docker network rm` | 删除网络 | `docker network rm mynet` |
| `docker network connect` | 把容器加入网络 | `docker network connect mynet myapp` |
| `docker network disconnect` | 把容器移出网络 | `docker network disconnect mynet myapp` |

---

## 七、宿主机

**宿主机 = 装 Docker 的那台电脑/服务器。**

类比：宿主机是房子，Docker 是隔断工具，容器是隔出来的小房间。

`-p 3306:3306` 左边是宿主机端口，右边是容器内部端口。外部想访问容器，必须通过宿主机的端口转进去。

---

## 八、部署 Java 项目

### 8.1 整体流程

打包 → 构建镜像 → 运行容器

### 8.2 具体步骤

```bash
# 1. 打包
mvn clean package -DskipTests

# 2. 构建镜像（Dockerfile 在 jar 包同级目录）
docker build -t docker-demo:1.0 .

# 3. 创建网络
docker network create hmall

# 4. 启动 MySQL
docker run -d --name mysql --network hmall -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123 -v mysql-data:/var/lib/mysql mysql

# 5. 启动 Java 项目
docker run -d --name hmall --network hmall -p 8080:8080 docker-demo:1.0

# 6. 看日志验证
docker logs -f hmall
```

### 8.3 注意事项

配置文件里的数据库连接要改成容器名：

```yaml
# 改之前（本地开发）
url: jdbc:mysql://localhost:3306/hmall

# 改之后（Docker 部署）
url: jdbc:mysql://mysql:3306/hmall
```

---

## 九、部署前端

### 9.1 前端本质

前端项目打包后就是静态文件（HTML/CSS/JS），用 Nginx 托管。

### 9.2 Dockerfile

```dockerfile
FROM nginx:latest
COPY dist/ /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

### 9.3 Nginx 配置（nginx.conf）

```nginx
server {
    listen       80;
    server_name  localhost;

    # 前端静态文件
    location / {
        root   /usr/share/nginx/html;
        index  index.html;
        try_files $uri $uri/ /index.html;  # 解决 Vue 路由刷新 404
    }

    # 反向代理后端接口
    location /api/ {
        proxy_pass http://hmall:8080/;
    }
}
```

### 9.4 前后端完整架构

```
浏览器 → Nginx(:80) → 前端页面
                   → /api/ → Java(:8080) → MySQL(:3306)
```

---

## 十、Docker Compose

### 10.1 为什么需要？

没有 Compose 时部署一个项目要跑 4 条 `docker run` 命令，参数一堆，容易打错。

**Docker Compose = 把一堆命令写成一个 YAML 文件，一条命令全部启动。**

### 10.2 docker-compose.yml 示例

```yaml
services:
  mysql:
    image: mysql
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      TZ: Asia/Shanghai
      MYSQL_ROOT_PASSWORD: 123
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - hmall

  hmall:
    build: .
    container_name: hmall
    ports:
      - "8080:8080"
    networks:
      - hmall
    depends_on:
      - mysql

  nginx:
    image: nginx
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./dist:/usr/share/nginx/html
    networks:
      - hmall
    depends_on:
      - hmall

networks:
  hmall:

volumes:
  mysql-data:
```

### 10.3 关键字段解读

| 字段 | 作用 | 对应命令 |
|---|---|---|
| `image` | 用哪个镜像 | `docker run` 最后面的镜像名 |
| `container_name` | 容器名 | `--name` |
| `ports` | 端口映射 | `-p` |
| `environment` | 环境变量 | `-e` |
| `volumes` | 数据卷/目录挂载 | `-v` |
| `networks` | 加入网络 | `--network` |
| `depends_on` | 启动顺序 | 手动控制先启动谁 |
| `build` | 用 Dockerfile 构建 | `docker build` |

### 10.4 常用命令

```bash
# 启动所有容器（后台运行）
docker compose up -d

# 停止所有容器
docker compose down

# 查看日志
docker compose logs -f

# 查看状态
docker compose ps

# 重新构建并启动（改了 Dockerfile 后用）
docker compose up -d --build
```

### 10.5 对比

| | 没有 Compose | 有 Compose |
|---|---|---|
| 启动命令 | 多条 `docker run` | 1 条 `docker compose up -d` |
| 网络 | 手动创建 | 自动创建 |
| 数据卷 | 手动挂载 | 自动管理 |
| 分享给别人 | 贴一堆命令 | 给一个 yml 文件 |

---

## 十一、常见问题

### Docker 里的 MySQL 和本地 MySQL 冲突吗？

**看端口。** 端口一样就冲突，换个端口就行。

```bash
# 本地 MySQL 占了 3306，Docker 就用 3307
docker run -d --name mysql -p 3307:3306 -e MYSQL_ROOT_PASSWORD=123 mysql
```

### Navicat 能连 Docker 里的 MySQL 吗？

**能。** Navicat 填宿主机 IP + 映射的端口即可。

| 字段 | 填什么 |
|---|---|
| 主机 | `localhost` |
| 端口 | 映射的宿主机端口（如 3307） |
| 用户名 | `root` |
| 密码 | 设置的密码 |

### Windows 本地能跑黑马课程的 Docker 部署吗？

**能。** Docker Desktop 在 Windows 上内置了 Linux 虚拟机，容器始终跑在 Linux 里，跟宿主机是 Windows 还是 CentOS 没区别。官方镜像（MySQL、Nginx、JDK）都是多架构支持的，直接用。

注意：挂载本地目录时路径要改成 Windows 格式：
```bash
# Linux
docker run -v /home/user/data:/var/lib/mysql ...
# Windows
docker run -v F:/data:/var/lib/mysql ...
```

---

## 十二、全文总结

| 概念 | 一句话解释 |
|---|---|
| 宿主机 | 装 Docker 的那台机器 |
| 容器网络 | 同一网络里的容器用容器名互相访问 |
| Dockerfile | 构建镜像的菜谱 |
| Docker Compose | 把一堆 docker run 写成一个 yml，一条命令启动 |
| 端口映射 | 宿主机端口:容器端口，外部通过宿主机访问容器 |
| 数据卷 | 容器的外挂硬盘，容器删了数据不丢 |

---

## 十三、关联笔记

- [[README|黑马商城项目总览]]

| 概念 | 一句话解释 |
|---|---|
| 镜像 | 模板，用来创建容器（类比：安装包） |
| 容器 | 镜像的运行实例（类比：装好的软件） |
| 数据卷 | 容器的外挂硬盘，容器删了数据不丢 |
| 端口映射 `-p` | 宿主机端口:容器端口，让外部能访问容器 |
| 环境变量 `-e` | 给镜像传配置参数 |

---

## 六、关联笔记

- [[README|黑马商城项目总览]]
