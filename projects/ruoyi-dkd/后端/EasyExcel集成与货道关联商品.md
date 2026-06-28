---
date: 2026-06-28
tags: [ruoyi-dkd, SpringBoot, MyBatis, EasyExcel]
---

# 【EasyExcel集成 + 货道关联商品】后端实现

## 一、写在前面

1. 为什么学这个：售货机业务需要批量导入商品（EasyExcel 替代 POI 解决内存问题），以及货道与商品的关联配置
2. 学完这篇你能掌握：EasyExcel 集成、DTO/VO 设计、批量更新 SQL、全局异常处理增强

## 二、EasyExcel 集成（需求文档 8.6 节）

### 2.1 为什么用 EasyExcel

| 对比项 | Apache POI | EasyExcel |
|---|---|---|
| 内存消耗 | 3M Excel 需要 100M 内存 | 3M Excel 只需几 M 内存 |
| 内存溢出 | 大文件容易 OOM | 不会出现内存溢出 |
| 使用复杂度 | 较复杂 | 更简单 |

### 2.2 集成步骤

**第一步：添加依赖** `dkd-common/pom.xml`

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>easyexcel</artifactId>
    <version>4.0.1</version>
</dependency>
```

**第二步：ExcelUtil 新增方法**

```java
// 导入
public List<T> importEasyExcel(InputStream is) throws Exception {
    return EasyExcel.read(is).head(clazz).sheet().doReadSync();
}

// 导出
public void exportEasyExcel(HttpServletResponse response, List<T> list, String sheetName) {
    EasyExcel.write(response.getOutputStream(), clazz).sheet(sheetName).doWrite(list);
}
```

**第三步：实体类添加注解** `Sku.java`

```java
@ExcelIgnoreUnannotated  // 忽略没有注解的字段
@ColumnWidth(16)         // 列宽
@HeadRowHeight(14)       // 表头行高
@HeadFontStyle(fontHeightInPoints = 11)  // 表头字体
public class Sku extends BaseEntity {
    @Excel(name = "商品名称")
    @ExcelProperty("商品名称")  // EasyExcel 注解
    private String skuName;
}
```

**第四步：Controller 改用 EasyExcel 方法**

```java
// 导入
List<Sku> skuList = util.importEasyExcel(file.getInputStream());
// 导出
util.exportEasyExcel(response, list, "商品管理数据");
```

### 2.3 Lombok 版本冲突

EasyExcel 4.0.1 和 Lombok 可能有兼容性问题，解决方案：
- VO/DTO 类手写 getter/setter，不用 `@Data`
- 项目中其他 VO 类（NodeVo、PartnerVo 等）都是手写的

## 三、货道关联商品（需求文档 8.7 节）

### 3.1 需求分析

对售货机内部的货道进行商品摆放管理，一个售货机有多个货道，每个货道可以放一个 SKU。

### 3.2 DTO 设计

```java
// 某个货道对应的 SKU 信息
public class ChannelSkuDto {
    private String innerCode;    // 售货机编号
    private String channelCode;  // 货道编号
    private Long skuId;          // 商品 ID
}

// 售货机货道配置
public class ChannelConfigDto {
    private String innerCode;              // 售货机编号
    private List<ChannelSkuDto> channelList;  // 货道配置列表
}
```

### 3.3 VO 设计（关联查询）

```java
public class ChannelVo extends Channel {
    private Sku sku;  // 关联的商品信息
}
```

Mapper XML 中使用 `<association>` 做关联查询：

```xml
<resultMap type="ChannelVo" id="ChannelVoResult">
    <!-- 货道字段映射 -->
    <association property="sku" javaType="Sku" 
                 column="sku_id" 
                 select="com.dkd.manage.mapper.SkuMapper.selectSkuBySkuId"/>
</resultMap>
```

### 3.4 核心业务逻辑

```java
public int setChannel(ChannelConfigDto channelConfigDto) {
    // 1. DTO 转 PO
    List<Channel> list = channelConfigDto.getChannelList().stream().map(c -> {
        Channel channel = channelMapper.getChannelInfo(c.getInnerCode(), c.getChannelCode());
        if (channel != null) {
            channel.setSkuId(c.getSkuId());
            channel.setUpdateTime(DateUtils.getNowDate());
        }
        return channel;
    }).collect(Collectors.toList());
    // 2. 批量修改货道
    return channelMapper.batchUpdateChannel(list);
}
```

### 3.5 批量更新 SQL

```xml
<update id="batchUpdateChannel" parameterType="java.util.List">
    <foreach item="channel" collection="list" separator=";">
        UPDATE tb_channel
        <set>
            <if test="channel.skuId != null">sku_id = #{channel.skuId},</if>
            <!-- 其他字段... -->
        </set>
        WHERE id = #{channel.id}
    </foreach>
</update>
```

**注意**：需要在 `application-druid.yml` 的 URL 中添加 `allowMultiQueries=true`，允许一次请求执行多条 SQL。

## 四、全局异常处理增强

在 `GlobalExceptionHandler` 中添加 `DataIntegrityViolationException` 处理：

```java
@ExceptionHandler(DataIntegrityViolationException.class)
public AjaxResult handelDataIntegrityViolationException(DataIntegrityViolationException e) {
    if (e.getMessage().contains("foreign")) {
        return AjaxResult.error("无法删除，有其他数据引用");
    }
    if (e.getMessage().contains("Duplicate")) {
        return AjaxResult.error("无法保存，名称已存在");
    }
    return AjaxResult.error("数据完整性异常，请联系管理员");
}
```

## 五、商品删除保护

删除商品前检查是否被货道关联：

```java
public int deleteSkuBySkuIds(Long[] skuIds) {
    int count = channelService.countChannelBySkuIds(skuIds);
    if (count > 0) {
        throw new ServiceException("此商品被货道关联，无法删除");
    }
    return skuMapper.deleteSkuBySkuIds(skuIds);
}
```

## 六、踩坑避坑指南

### 逻辑坑

**坑 1：Lombok @Data 不生效**
- 问题：EasyExcel 4.0.1 和 Lombok 版本冲突
- 解决：VO/DTO 类手写 getter/setter

**坑 2：批量更新 SQL 报错**
- 报错：`Multi-statement not allowed`
- 解决：URL 添加 `allowMultiQueries=true`

## 七、代码变更

- 修改：`dkd-common/pom.xml` — 添加 EasyExcel 依赖
- 修改：`ExcelUtil.java` — 添加 importEasyExcel、exportEasyExcel 方法
- 修改：`Sku.java` — 添加 @ExcelProperty 注解
- 修改：`SkuController.java` — 改用 EasyExcel 方法
- 新增：`ChannelSkuDto.java` — 货道 SKU DTO
- 新增：`ChannelConfigDto.java` — 货道配置 DTO
- 修改：`ChannelVo.java` — 去掉 @Data，手写 getter/setter
- 修改：`ChannelMapper.java` — 添加 getChannelInfo、batchUpdateChannel、selectChannelVoListByInnerCode
- 修改：`ChannelMapper.xml` — 添加关联查询和批量更新 SQL
- 修改：`IChannelService.java` — 添加 setChannel、selectChannelVoListByInnerCode
- 修改：`ChannelServiceImpl.java` — 实现 setChannel、selectChannelVoListByInnerCode
- 修改：`ChannelController.java` — 添加 PUT /config 接口
- 修改：`GlobalExceptionHandler.java` — 添加 Duplicate 异常处理
- 修改：`SkuServiceImpl.java` — 删除前检查货道关联

## 八、关联笔记

- [[项目概述]]
