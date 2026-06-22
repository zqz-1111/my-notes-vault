---
date: '2026-06-22'
tags:
  - hm-dianping
  - Redis
  - GEO
  - 附近商户
---

# 【Redis GEO】附近商户功能实现

## 一、写在前面

1. 为什么学这个：很多本地生活 App 都有"附近美食"、"附近 KTV"功能，本质就是**根据用户坐标搜索附近 N 公里的商户**，按距离排序返回
2. 学完这篇你能掌握：Redis GEO 数据结构的使用、GEORADIUS 查询、分页方案、前后端距离显示

## 二、前置准备

- Redis 3.2+（GEO 功能从 3.2 开始支持）
- Spring Boot 2.3（用 `GEORADIUS`，2.6+ 才有 `GEOSEARCH`）
- 数据库 `tb_shop` 表有 `x`（经度）和 `y`（纬度）字段

## 三、Redis GEO 是什么

GEO（Geographic）是 Redis 3.2 引入的**地理位置数据结构**，底层其实是 **Sorted Set**：

- **member**：位置名称（我们存商户 ID）
- **score**：geohash 编码值（把二维经纬度编码成一维数字，方便范围查询）

核心命令：

| 命令 | 作用 | 等价于 |
|------|------|--------|
| `GEOADD key 经度 纬度 member` | 添加位置 | `ZADD key geohash_score member` |
| `GEORADIUS key 经度 纬度 半径 km` | 搜索附近 | 按 geohash 范围查询 |
| `GEODIST key member1 member2` | 两点距离 | 计算球面距离 |
| `GEOPOS key member` | 获取坐标 | 反查经纬度 |

## 四、实现步骤

### 4.1 第一步：数据预热（写入 Redis）

**为什么按类型分组？** 前端传 `typeId` 参数，如果所有商户混在一个 GEO key 里，每次查询都要搜全部数据再过滤。按类型分组后，直接搜 `shop:geo:{typeId}`，精准命中。

```java
public void saveShop2Redis() {
    // 1. 查出所有商铺
    List<Shop> list = list();
    // 2. 按 typeId 分组
    Map<Long, List<Shop>> map = list.stream()
        .collect(Collectors.groupingBy(Shop::getTypeId));
    // 3. 遍历每个类型，批量写入 GEO
    for (Map.Entry<Long, List<Shop>> entry : map.entrySet()) {
        Long typeId = entry.getKey();
        List<Shop> shops = entry.getValue();
        String key = SHOP_GEO_KEY + typeId;  // "shop:geo:1"
        
        // 用 pipeline 批量写入，减少网络往返
        List<RedisGeoCommands.GeoLocation<String>> locations = new ArrayList<>();
        for (Shop shop : shops) {
            locations.add(new RedisGeoCommands.GeoLocation<>(
                shop.getId().toString(),
                new Point(shop.getX(), shop.getY())
            ));
        }
        stringRedisTemplate.opsForGeo().add(key, locations);
    }
}
```

**两种批量写入方式对比：**

| 方式 | 代码 | 优缺点 |
|------|------|--------|
| pipeline | `executePipelined(connection -> { geoAdd... })` | 底层管道，性能好 |
| GeoLocation 列表 | `opsForGeo().add(key, locations)` | 代码简洁，推荐 |

**执行后 Redis 数据结构：**
```
shop:geo:1 (Sorted Set) → {
  "1"  → geohash(120.149, 30.316)   // 103茶餐厅
  "4"  → geohash(120.147, 30.313)   // Mamala
  "5"  → geohash(120.158, 30.311)   // 海底捞
  ...
}
shop:geo:2 → {
  "10" → geohash(120.149, 30.325)   // 开乐迪KTV
  ...
}
```

### 4.2 第二步：查询附近商户

```java
public Result queryNearby(Integer typeId, Integer current, Double x, Double y) {
    // 1. 没有坐标时降级为普通分页查询
    if (x == null || y == null) {
        Page<Shop> page = query()
                .eq("type_id", typeId)
                .page(new Page<>(current, SystemConstants.DEFAULT_PAGE_SIZE));
        return Result.ok(page.getRecords());
    }

    // 2. 计算分页参数
    int from = (current - 1) * SystemConstants.DEFAULT_PAGE_SIZE;
    int end = current * SystemConstants.DEFAULT_PAGE_SIZE;

    // 3. GEORADIUS 查询附近 5km 的商户
    String key = SHOP_GEO_KEY + typeId;
    Circle circle = new Circle(new Point(x, y), new Distance(5, Metrics.KILOMETERS));
    GeoResults<RedisGeoCommands.GeoLocation<String>> results = stringRedisTemplate
            .opsForGeo()
            .radius(key, circle,
                    RedisGeoCommands.GeoRadiusCommandArgs.newGeoRadiusArgs()
                            .includeDistance()   // 返回距离值
                            .sortAscending()     // 按距离升序
                            .limit(end)          // 取前 end 个
            );
    if (results == null || results.getContent().isEmpty()) {
        return Result.ok(Collections.emptyList());
    }

    // 4. 截取当前页 from ~ end
    List<GeoResult<RedisGeoCommands.GeoLocation<String>>> list = results.getContent();
    if (list.size() <= from) {
        return Result.ok(Collections.emptyList());
    }
    List<Long> ids = new ArrayList<>();
    Map<String, Distance> distanceMap = new HashMap<>();
    list.stream().skip(from).forEach(result -> {
        String shopIdStr = result.getContent().getName();
        ids.add(Long.valueOf(shopIdStr));
        distanceMap.put(shopIdStr, result.getDistance());
    });

    // 5. 查数据库，用 ORDER BY FIELD 保持 Redis 的距离排序
    String idStr = StrUtil.join(",", ids);
    List<Shop> shops = query()
            .in("id", ids)
            .last("ORDER BY FIELD(id," + idStr + ")")
            .list();
    for (Shop shop : shops) {
        shop.setDistance(distanceMap.get(shop.getId().toString()).getValue());
    }
    return Result.ok(shops);
}
```

**核心逻辑拆解：**

#### GEORADIUS 命令
```
GEORADIUS shop:geo:1 120.149 30.316 5 km WITHDIST ASC COUNT 10
```
- `120.149 30.316` — 用户当前坐标（圆心）
- `5 km` — 搜索半径
- `WITHDIST` — 返回结果带上距离
- `ASC` — 按距离升序（由近到远）
- `COUNT 10` — 最多返回 10 个

#### 分页方案：limit(end) + skip(from)

Redis GEO 不支持 offset 分页，所以：
1. 用 `limit(end)` 一次性取到当前页所有数据（比如第 2 页，end=20，取前 20 个）
2. 用 `stream().skip(from)` 跳过前面的（跳过前 10 个，取第 11~20 个）

#### ORDER BY FIELD 保序

```sql
SELECT * FROM tb_shop WHERE id IN (6,4,8,1,5) ORDER BY FIELD(id,6,4,8,1,5)
```

**为什么需要这个？** MyBatis-Plus 的 `listByIds` 查出来顺序是数据库默认顺序（通常是主键升序），不是 Redis 返回的距离顺序。`ORDER BY FIELD(id,6,4,8,1,5)` 强制按传入的 ID 顺序排列，保持距离排序。

## 五、踩坑避坑指南

### 坑 1：GEORADIUS 的 sortAscending() 不生效

- 问题描述：Spring Boot 2.3 的 `GEORADIUS` 命令，`sortAscending()` 参数有时不生效，返回的数据顺序是乱的
- 解决方案：在 Java 端用 `shopIds.sort((a, b) -> Double.compare(distanceMap.get(a), distanceMap.get(b)))` 手动排序兜底

### 坑 2：数据库查询不保序

- 问题描述：Redis 返回的距离顺序是对的，但 `listByIds(ids)` 查出来顺序乱了
- 后果：前端显示的距离顺序和实际距离不一致（1.0km, 1.7km, 1.0km, 2.0km）
- 正确做法：用 `ORDER BY FIELD(id, idStr)` 保持 Redis 的排序

### 坑 3：Spring Boot 版本不支持 GEOSEARCH

- 问题描述：`GEOSEARCH` 是 Redis 6.2 + Spring Boot 2.6+ 才有的命令
- 报错信息：`无法解析符号 'GeoReference'`、`无法解析 'GeoOperations' 中的方法 'search'`
- 解决方案：降级用 `GEORADIUS`（旧版 API），用 `Circle` + `radius()` 替代 `GeoReference` + `search()`

### 坑 4：前端滚动加载不触发

- 问题描述：店铺卡片太矮，5 条数据没超出容器高度，`@scroll` 事件根本没触发
- 解决方案：增大 `DEFAULT_PAGE_SIZE`（5→10），让内容撑满容器

### 坑 5：浏览器缓存导致数据不更新

- 问题描述：改了前端代码，刷新页面没效果
- 解决方案：`Ctrl + F5` 强制刷新，清掉浏览器缓存

## 六、效果验证

```bash
# 1. 初始化数据
curl -X POST http://localhost:8081/shop/geo/init

# 2. 查询附近美食（typeId=1）
curl "http://localhost:8081/shop/of/nearby?typeId=1&current=1&x=120.149993&y=30.334229"

# 3. 返回结果（按距离升序）
# 蔡馬洪涛烤肉     - 0.17km
# 羊老三羊蝎子     - 1.00km
# 浅草屋寿司       - 1.00km
# 新白鹿餐厅       - 1.05km
# 幸福里老北京涮锅  - 1.74km
```

## 七、全文总结

1. **Redis GEO 底层是 Sorted Set**，score 是 geohash 编码，支持距离计算和范围查询
2. **按类型分组存储**（`shop:geo:{typeId}`），查询时精准命中，不用全量扫描
3. **分页用 limit(end) + skip(from)**，因为 GEO 不支持 offset
4. **ORDER BY FIELD 保序**，解决数据库查询打乱 Redis 距离排序的问题
5. **Spring Boot 2.3 只能用 GEORADIUS**，2.6+ 才有 GEOSEARCH

## 八、关联笔记

- [[Redis缓存三大经典问题]]
- [[分布式锁原理详解]]
- [[SpringBoot常用注解大全]]

## 九、代码变更

- 新增：`ShopController.java` — `/shop/of/nearby` 接口 + `/shop/geo/init` 初始化接口
- 新增：`IShopService.java` — `queryNearby()` + `saveShop2Redis()` 方法签名
- 修改：`ShopServiceImpl.java` — 实现 `queryNearby()` 和 `saveShop2Redis()`
- 修改：`SystemConstants.java` — `DEFAULT_PAGE_SIZE` 从 5 改为 10
