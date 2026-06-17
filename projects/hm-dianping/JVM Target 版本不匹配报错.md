---
date: 2026-06-17
tags: [踩坑, IDEA, Java, hm-dianping]
---

# JVM Target 版本不匹配报错

## 问题描述

启动项目时报错：

```
java: 无法编译为 JVM 目标 17 配置的模块 'hm-dianping': 指定的回退 SDK 版本 8 不支持所需 jvm 目标 17
```

本地装的是 **JDK 8**（Dragonwell 1.8），但 IDEA 认为模块需要 target 17。

## 原因分析

JDK 8 最高只能编译到 target 1.8，不支持 target 17。

pom.xml 里 `java.version` 配置的是 1.8，没问题。问题出在 **IntelliJ IDEA 的内部设置**被改成了 17（可能是某个依赖或插件自动改的）。

## 解决方案

在 IntelliJ IDEA 里改 3 个地方：

### 1. 模块 Language Level

`File → Project Structure → Modules` → 选 `hm-dianping` → **Language Level** 改成 `8`

### 2. 项目 SDK 和 Language Level

`File → Project Structure → Project`
- **Project SDK** 确认是 JDK 8
- **Project language level** 选 `8`

### 3. Java Compiler Target

`File → Settings → Build, Execution, Deployment → Compiler → Java Compiler` → **Target bytecode version** 改成 `8`

改完后 `Build → Rebuild Project` 重新编译。

## 关联笔记

- [[hm-dianping 项目搭建]]
