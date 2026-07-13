---
date: 2026-07-13
tags: [Elasticsearch, IK分词器, 倒排索引, 黑马商城]
---

# 【Elasticsearch】入门：倒排索引、核心概念与 IK 分词器

## 一、写在前面

1. 为什么学 Elasticsearch？—— MySQL 的 `LIKE '%手机%'` 不走索引，数据量大时查询性能断崖式下降。ES 基于倒排索引，百万级数据搜索依然毫秒级响应
2. 学完这篇你能掌握：ES 核心概念（文档/索引/映射）、倒排索引原理、IK 分词器安装与使用、文档 CRUD 基本操作

## 二、前置准备

- 虚拟机已安装 Docker，网络 `hm-net` 已创建
- ES 版本：7.12.1
- 内核参数：`vm.max_map_count=262144`（ES 启动必须）

## 三、ELK 技术栈简介

ELK 是 Elastic 公司的一套搜索引擎技术栈：

| 组件 | 作用 |
|---|---|
| **Elasticsearch** | 数据存储、搜索、分析（核心） |
| **Logstash / Beats** | 数据收集 |
| **Kibana** | 可视化控制台，提供 DevTools 操作 ES |

学习重点是 Elasticsearch 本身。

## 四、Docker 部署 ES + Kibana

### 4.1 设置内核参数

```bash
# 临时生效
sysctl -w vm.max_map_count=262144

# 永久生效
echo "vm.max_map_count=262144" >> /etc/sysctl.conf
sysctl -p
```

**为什么要设？** ES 底层用 `mmap` 读取索引文件，默认 65530 太小，启动直接报错。

### 4.2 启动 ES

```bash
docker run -d \
  --name es \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  -e "discovery.type=single-node" \
  -v es-data:/usr/share/elasticsearch/data \
  -v es-plugins:/usr/share/elasticsearch/plugins \
  --privileged \
  --network hm-net \
  -p 9200:9200 \
  -p 9300:9300 \
  elasticsearch:7.12.1
```

| 参数 | 说明 |
|---|---|
| `ES_JAVA_OPTS` | JVM 堆内存，生产建议 1G+ |
| `discovery.type=single-node` | 单节点模式，不加入集群 |
| `es-data` | 数据卷，持久化索引数据 |
| `es-plugins` | 插件卷，存放 IK 分词器等插件 |

### 4.3 启动 Kibana

```bash
docker run -d \
  --name kibana \
  -e "ELASTICSEARCH_HOSTS=http://es:9200" \
  --network hm-net \
  -p 5601:5601 \
  kibana:7.12.1
```

**版本必须一致**，否则可能不兼容。

### 4.4 访问地址

- ES：`http://192.168.188.130:9200`
- Kibana：`http://192.168.188.130:5601` → 左侧菜单 DevTools

### 4.5 Docker 开机自启

```bash
docker update --restart always mysql nacos mq sentinel seata es kibana
```

验证：
```bash
docker inspect --format '{{.Name}}: {{.HostConfig.RestartPolicy.Name}}' $(docker ps -aq)
```

## 五、倒排索引原理

### 5.1 正向索引（MySQL）

MySQL 的 B+ 树索引是正向索引：**文档 → 词条**。

```sql
SELECT * FROM tb_goods WHERE title LIKE '%手机%';
```

- 索引字段精确匹配 → 走索引，快
- 模糊匹配 → 全表扫描，逐行判断，数据越多越慢

### 5.2 倒排索引（ES）

倒排索引反过来：**词条 → 文档**。

**写入阶段（建索引时）：**
```
文档1: "华为手机"  → 分词 → ["华为", "手机"]
文档2: "苹果手机"  → 分词 → ["苹果", "手机"]
文档3: "华为电脑"  → 分词 → ["华为", "电脑"]

建好的倒排索引表：
  "华为" → [1, 3]
  "手机" → [1, 2]
  "苹果" → [2]
  "电脑" → [3]
```

**查询阶段（搜索时）：**
```
搜"华为手机" → 分词 → ["华为", "手机"]
查表："华为" → [1, 3]
查表："手机" → [1, 2]
取交集 → [1]  ← 文档1同时包含两个词
```

### 5.3 对比总结

| | 正向索引（MySQL） | 倒排索引（ES） |
|---|---|---|
| **数据结构** | 文档 → 包含哪些词 | 词 → 在哪些文档里 |
| **查询方式** | 全表扫描逐行匹配 | 查表拿文档ID，再查正向索引 |
| **性能** | 数据越多越慢 | 数据量对性能影响小 |
| **本质** | 查的时候才匹配 | 写入时就建好匹配关系（空间换时间） |

## 六、ES 核心概念

### 6.1 概念对照表

| MySQL | Elasticsearch | 说明 |
|---|---|---|
| Table | Index（索引） | 文档的集合，类似数据库的表 |
| Row | Document（文档） | 一条数据，JSON 格式 |
| Column | Field（字段） | JSON 文档中的字段 |
| Schema | Mapping（映射） | 字段的类型约束，类似表结构 |
| SQL | DSL | ES 的 JSON 风格查询语句 |

### 6.2 文档（Document）

- ES 面向文档存储，数据序列化为 JSON
- 一行数据 = 一个 JSON 文档
- 文档大小一般几 KB 到几百 KB，不适合存大文本

### 6.3 索引（Index）

- 同类文档的集合，类似数据库的"表"
- 例如：商品索引、用户索引、订单索引

### 6.4 映射（Mapping）

- 定义索引中字段的名称、类型、约束
- 类似数据库的表结构定义

### 6.5 MySQL vs ES 适用场景

| 场景 | 推荐 |
|---|---|
| 事务操作、数据安全 | MySQL |
| 海量数据搜索、分析 | Elasticsearch |
| 企业实践 | MySQL 做主库 + ES 做搜索引擎，数据同步 |

## 七、IK 分词器

### 7.1 为什么需要 IK？

ES 默认的 Standard 分词器对中文是逐字切分（"我是中国人" → "我"、"是"、"中"、"国"、"人"），没有语义。IK 分词器是中文分词的标准方案。

### 7.2 两种模式

| 模式 | 说明 | 示例 |
|---|---|---|
| `ik_smart` | 智能切分，粗粒度 | "我是中国人" → 我、是、中国人 |
| `ik_max_word` | 最细切分，细粒度 | "我是中国人" → 我、是、中国人、中国、国人 |

- **创建索引时**推荐用 `ik_max_word`（细粒度，多建词条，搜索更全）
- **搜索时**推荐用 `ik_smart`（粗粒度，快速匹配）

### 7.3 安装 IK 分词器

GitHub 被墙，手动安装：

```bash
# 1. 本地下载 zip：https://github.com/medcl/elasticsearch-analysis-ik/releases/tag/v7.12.1
# 2. 上传到宿主机数据卷
#    /var/lib/docker/volumes/es-plugins/_data/

# 3. 复制到容器 /tmp 并安装
docker cp /var/lib/docker/volumes/es-plugins/_data/elasticsearch-analysis-ik-7.12.1.zip es:/tmp/
docker exec -it es ./bin/elasticsearch-plugin install file:///tmp/elasticsearch-analysis-ik-7.12.1.zip

# 4. 删掉 zip，重启
docker restart es
```

**踩坑：** zip 不能放在 plugins 目录下，ES 会把它当成已安装的插件报错。

### 7.4 扩展词典

IK 默认不认识的词会被拆错。在 `IKAnalyzer.cfg.xml` 中配置扩展词典：

```xml
<entry key="ext_dict">ext.dic</entry>
```

`ext.dic` 文件中每行一个词，例如：
```
黑马商城
传智播黑
```

一般业务场景先不加，等发现分词不准时再按需扩展。

### 7.5 验证分词器

Kibana DevTools 中执行：

```json
POST _analyze
{
  "analyzer": "ik_smart",
  "text": "我是中国人"
}
```

返回 tokens 列表即说明安装成功。

## 八、全文总结

1. **倒排索引**是 ES 高性能的核心：写入时分词建表，查询时直接查表，空间换时间
2. **核心概念**：Index=表、Document=行、Field=列、Mapping=表结构
3. **IK 分词器**两种模式：`ik_smart`（粗粒度）和 `ik_max_word`（细粒度）
4. **MySQL + ES 配合使用**：MySQL 保事务安全，ES 保搜索性能，数据同步

## 九、关联笔记

- [[README]]
- [[../09-RabbitMQ/RabbitMQ入门|RabbitMQ入门]]

## 十、代码变更

- 无代码变更，本篇为理论 + 环境搭建
