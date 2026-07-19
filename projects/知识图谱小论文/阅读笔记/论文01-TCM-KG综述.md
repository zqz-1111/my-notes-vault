---
date: 2026-07-18
tags: [知识图谱, 论文, 综述, 中医药]
---

# 论文01：TCM KG 综述

## 基本信息

- **标题**：A Review of Knowledge Graph in Traditional Chinese Medicine: Analysis, Construction, Application and Prospects
- **来源**：SciOpen / CMC，2024年12月
- **作者**：Xiaolong Qu 等（北京林业大学 + 中国中医科学院）

## 核心内容

### 四大构建技术

1. **知识表示**：本体表示（RDF/OWL）+ 知识表示学习（Embedding/GCN）
2. **知识抽取**：规则/词典 → CRF → BiLSTM-CRF → BERT-BiLSTM-CRF
3. **知识融合**：实体对齐（一物多名）+ 实体消歧（多义词）
4. **知识推理**：逻辑规则推理 + 分布式表示推理 + 神经网络推理

### 七大子领域

基本理论、处方、中药、证候、临床实践、养生、其他

### 五大应用

信息检索、辅助诊疗、智能问答、推荐、知识挖掘

## 对我们的启发

- 综述指出"将中药药理学特征作为独立模态融入KG"是未来方向
- 现有挑战：缺乏大规模标准化标注语料、嵌套实体难处理、多模态信息未融合
- 技术演进路线：规则 → CRF → BiLSTM-CRF → BERT-BiLSTM-CRF → **我们的改进**
