---
date: 2026-07-18
tags: [知识图谱, 论文, 古籍, 多智能体]
---

# 论文15：古籍 KG + 多智能体

## 基本信息

- **标题**：Integrating KG with Ancient Chinese Medicine Classics: Challenges and Future Prospects of MAS Convergence
- **来源**：Chinese Medicine（Springer），2025年10月，医学一区/中医学顶刊

## 核心方案

MAS（多智能体）+ RAG（检索增强）+ KG 协同架构

### 技术路线

顶层本体设计 → 多Agent流水线（翻译/对齐/NER/RE/校验）→ Neo4j存储

### Agent 分工

- 翻译Agent：古文→白话文
- 同义词Agent：实体归一化
- NER Agent：命名实体识别
- RE Agent：关系抽取
- 校验Agent：逻辑一致性验证

### RAG 增强

遇到歧义时检索历史注解、专业字典，作为上下文佐证

## 古籍文本五大难点

1. 缺乏统一术语系统
2. 文法结构灵活多变（倒装、省略）
3. 修辞手法复杂（互文、隐喻）
4. 同义词/文本相似混淆严重
5. 复杂嵌套实体结构

## 现有技术 baseline

- 词向量：Word2Vec / BERT / ALBERT
- NER：BERT-BiLSTM-CRF
- 关系分类：朴素贝叶斯 / 规则模板
- 图存储：Neo4j

## 三个局限 = 我们的创新切入点

1. **翻译依赖症** → 我们直接在原文上增强，不依赖翻译
2. **大模型幻觉 vs 低容错率** → 我们用轻量BERT作确定性模型
3. **嵌套实体识别乏力** → 我们融合KG先验知识解决

## 创新点调整

原方案：融合药理学特征的BERT RE
新方案：**融合药理学特征 + KG结构先验的BERT，提升中医嵌套实体识别和关系抽取准确率**
