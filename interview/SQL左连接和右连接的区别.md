---
date: 2026-06-26
tags: [interview, SQL, MySQL]
---

# SQL 左连接和右连接的区别

## 问题

SQL 中 LEFT JOIN 和 RIGHT JOIN 有什么区别？分别在什么场景下使用？

## 答案

**核心区别：以哪张表为基准保留数据**

| 连接类型 | 基准表 | 说明 |
|---|---|---|
| `LEFT JOIN` | 左表（写在前面的表） | 保留左表所有记录，右表没匹配到就填 NULL |
| `RIGHT JOIN` | 右表（写在后面的表） | 保留右表所有记录，左表没匹配到就填 NULL |

### 举例说明

**区域表 tb_region**（3条数据）
| id | region_name |
|---|---|
| 1 | 朝阳区 |
| 2 | 海淀区 |
| 3 | 东城区 |

**点位表 tb_node**（2条数据，东城区没有点位）
| id | node_name | region_id |
|---|---|---|
| 1 | 三里屯 | 1 |
| 2 | 五道口 | 2 |

**LEFT JOIN**（以左表为准）：
```sql
SELECT r.id, r.region_name, COUNT(n.id) as node_count
FROM tb_region r
LEFT JOIN tb_node n ON r.id = n.region_id
GROUP BY r.id;
```
结果：保留所有区域，没点位的显示 0
| id | region_name | node_count |
|---|---|---|
| 1 | 朝阳区 | 1 |
| 2 | 海淀区 | 1 |
| 3 | 东城区 | 0 ← 保留了 |

**RIGHT JOIN**（以右表为准）：
```sql
SELECT r.id, r.region_name, COUNT(n.id) as node_count
FROM tb_region r
RIGHT JOIN tb_node n ON r.id = n.region_id
GROUP BY r.id;
```
结果：只保留有点位的区域，东城区丢失了

## 深入追问

- 追问 1：LEFT JOIN 和 INNER JOIN 的区别？
  - INNER JOIN 只返回两表都匹配的记录，LEFT JOIN 保留左表所有记录
- 追问 2：什么时候用 RIGHT JOIN？
  - 实际开发中很少用，一般把表顺序调换用 LEFT JOIN 代替，可读性更好
- 追问 3：LEFT JOIN 后面的 WHERE 条件和 ON 条件有什么区别？
  - ON 条件在连接时过滤，WHERE 条件在连接后过滤

## 来源

ruoyi-dkd 项目 — 区域管理改造中，查询区域列表需要显示每个区域的点位数，使用 LEFT JOIN 实现关联查询
