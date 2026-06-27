---
date: 2026-06-27
tags: [ruoyi-dkd, MinIO, 文件存储, 后端]
---

# 【文件存储】MinIO 集成替代本地磁盘存储

## 一、写在前面

1. **解决什么问题**：若依框架默认把上传的文件存到服务器本地磁盘（`D:/ruoyi/uploadPath`），有硬件瓶颈、单点故障、性能差等问题。需求文档 5.4 要求用对象存储替代。
2. **学完你能掌握**：在 Spring Boot 项目中集成 MinIO 对象存储，改造通用上传接口，实现文件上传到 MinIO 并返回可访问的 URL。

## 二、前置准备

- MinIO 服务已启动（`localhost:9000`）
- 已创建 bucket（本项目用 `sky-take-out`）
- bucket 访问策略设为 public（否则上传成功但图片不显示）

## 三、实现步骤

### 3.1 添加 Maven 依赖

`dkd-common/pom.xml` 加 minio 依赖：

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.7</version>
</dependency>
```

### 3.2 解决 OkHttp 版本冲突

Spring Boot 2.5.15 拉了 OkHttp 3.14.9，但 MinIO 8.5.7 需要 OkHttp 4.x。

**解法**：在根 `pom.xml` 的 `<dependencyManagement>` 里强制覆盖版本：

```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.10.0</version>
</dependency>
```

### 3.3 添加配置

`application.yml`：

```yaml
minio:
  endpoint: http://localhost:9000
  access-key: admin
  secret-key: 12345678
  bucket-name: sky-take-out
```

### 3.4 配置类 MinioConfig

```java
@Configuration
@ConfigurationProperties(prefix = "minio")
public class MinioConfig {
    private String endpoint;
    private String accessKey;
    private String secretKey;
    private String bucketName;
    // getter/setter...

    @Bean
    public MinioClient minioClient() {
        return MinioClient.builder()
                .endpoint(endpoint)
                .credentials(accessKey, secretKey)
                .build();
    }
}
```

### 3.5 工具类 MinioUtils

```java
@Component
public class MinioUtils {
    @Autowired
    private MinioClient minioClient;
    @Autowired
    private MinioConfig minioConfig;

    public String upload(MultipartFile file) throws Exception {
        // 生成文件名：yyyy/MM/dd/UUID.后缀
        String extension = file.getOriginalFilename()
            .substring(file.getOriginalFilename().lastIndexOf("."));
        String fileName = DateUtils.datePath() + "/"
            + UUID.randomUUID().toString().replace("-", "") + extension;

        // 上传到MinIO
        try (InputStream inputStream = file.getInputStream()) {
            minioClient.putObject(
                PutObjectArgs.builder()
                    .bucket(minioConfig.getBucketName())
                    .object(fileName)
                    .stream(inputStream, file.getSize(), -1)
                    .contentType(file.getContentType())
                    .build()
            );
        }
        // 返回完整URL
        return minioConfig.getEndpoint() + "/"
            + minioConfig.getBucketName() + "/" + fileName;
    }
}
```

### 3.6 改造 CommonController

```java
@Autowired
private MinioUtils minioUtils;

@PostMapping("/upload")
public AjaxResult uploadFile(MultipartFile file) throws Exception {
    try {
        String url = minioUtils.upload(file);
        AjaxResult ajax = AjaxResult.success();
        ajax.put("url", url);
        ajax.put("fileName", url);  // 关键：fileName要返回完整URL
        ajax.put("newFileName", url.substring(url.lastIndexOf("/") + 1));
        ajax.put("originalFilename", file.getOriginalFilename());
        return ajax;
    } catch (Exception e) {
        return AjaxResult.error(e.getMessage());
    }
}
```

## 四、踩坑避坑

### 报错坑：OkHttp 版本冲突

- **报错信息**：`The method okhttp3.RequestBody.create([BLokhttp3/MediaType;) does not exist`
- **原因**：MinIO 8.5.7 需要 OkHttp 4.x 的 `RequestBody.create(byte[], MediaType)` 方法签名，但 SpringBoot BOM 锁定了 OkHttp 3.14.9
- **解决**：根 pom 的 `<dependencyManagement>` 强制指定 OkHttp 4.10.0

### 报错坑：Lombok @Data 不生效

- **报错信息**：编译通过但运行时找不到 getter/setter
- **原因**：项目的 maven-compiler-plugin 版本太旧（3.1），Lombok 注解处理器没正确工作
- **解决**：Domain 类统一手写 getter/setter，不用 Lombok

### 逻辑坑：MinIO 文件不显示

- **问题**：上传成功，返回 URL 也正确，但浏览器直接访问 403
- **原因**：MinIO bucket 默认是私有的
- **解决**：在 MinIO 控制台把 bucket 的 Access Policy 改为 public

### 逻辑坑：前端图片不显示

- **问题**：MinIO URL 浏览器能访问，但前端页面不显示
- **原因**：前端 `ImageUpload` 组件用 `res.fileName` 作为显示 URL，我们返回的是原始文件名而不是完整 URL
- **解决**：`fileName` 字段返回完整的 MinIO URL（和 `url` 一样）

## 五、涉及文件

| 文件 | 操作 | 说明 |
|---|---|---|
| `pom.xml`（根） | 修改 | 强制 OkHttp 4.10.0 |
| `dkd-common/pom.xml` | 修改 | 加 minio 依赖 |
| `application.yml` | 修改 | 加 minio 配置 |
| `MinioConfig.java` | 新建 | 配置类 + MinioClient Bean |
| `MinioUtils.java` | 新建 | 上传工具类 |
| `CommonController.java` | 修改 | 上传接口改用 MinIO |

## 六、总结

- MinIO 是自建对象存储的最佳选择，兼容 S3 协议
- 核心流程：配置类创建 MinioClient → 工具类封装上传 → Controller 调用
- `fileName` 返回完整 URL 是关键，前端组件依赖这个字段显示图片
- 通用上传接口改一次，所有业务模块上传图片都能用

## 七、关联笔记

- [[人员管理新增修改补充冗余字段]] — 同一个项目的后端改造
- [[需求文档阅读与代码改造]] — 需求文档 5.4 文件存储章节

## 八、代码变更

- 新增：`dkd-common/.../config/MinioConfig.java` — MinIO 配置类
- 新增：`dkd-common/.../utils/file/MinioUtils.java` — MinIO 上传工具类
- 修改：`dkd-admin/.../controller/common/CommonController.java` — 上传接口改用 MinIO
- 修改：`dkd-common/pom.xml` — 加 minio 依赖
- 修改：`pom.xml`（根） — 强制 OkHttp 4.10.0
- 修改：`application.yml` — 加 minio 配置
