---
date: 2026-07-14
tags: [snippets, Lombok, Maven, 踩坑]
---

# Lombok 与 Java 版本踩坑指南

## 用途

Maven 编译时 Lombok getter/setter 没生成，报"找不到符号"错误。

## 问题现象

```
java: 找不到符号
  符号: 方法 getKey()
  位置: 类型为ItemPageQuery的变量 query
```

## 根因分析

Lombok 注解处理器跟 JDK 版本不兼容。常见场景：
- 系统装了多个 JDK（比如 JDK 11 和 JDK 25）
- 终端的 `JAVA_HOME` 指向了高版本 JDK
- IDEA 里配了 JDK 11 一切正常，但终端编译用的是另一个 JDK

```bash
# 检查当前 Java 版本
java -version
# 检查 JAVA_HOME
echo $JAVA_HOME
```

## 解决方案

### 方案 1：指定 JAVA_HOME（推荐）

终端编译前先切到正确的 JDK：

```bash
# Windows Git Bash
export JAVA_HOME="/e/jdk-11.0.31+11"
export PATH="$JAVA_HOME/bin:$PATH"
```

### 方案 2：升级 Lombok 版本

在父 pom.xml 改版本：

```xml
<org.projectlombok.version>1.18.36</org.projectlombok.version>
```

### 方案 3：显式配置注解处理器

在父 pom.xml 的 `maven-compiler-plugin` 加 `annotationProcessorPaths`：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <source>11</source>
        <target>11</target>
        <annotationProcessorPaths>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>1.18.36</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

**注意：改完要 `mvn clean install`，旧的 .class 文件会缓存！**

## 额外踩坑：settings.xml 覆盖项目配置

Maven 的 `settings.xml` 里如果有 `<maven.compiler.release>17</maven.compiler.release>`，会覆盖项目 pom 里的 Java 11 配置，导致 JDK 11 编译不了。

```xml
<!-- settings.xml 里的这个会覆盖项目配置，删掉！ -->
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <maven.compiler.release>17</maven.compiler.release>
</properties>
```

## 使用场景

- 新建 Maven 模块编译报 Lombok 相关错误
- 终端编译和 IDEA 编译结果不一致
- 多人协作时不同人的 JDK 版本不同
