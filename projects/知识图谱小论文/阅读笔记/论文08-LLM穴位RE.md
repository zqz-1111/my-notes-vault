---
date: 2026-07-21
tags: [知识图谱, RE, LLM, GPT, 针灸, Prompt工程]
---

# 论文8：LLM做关系抽取 — 针灸穴位案例

## 基本信息

- **标题**：Relation Extraction Using Large Language Models: A Case Study on Acupuncture Point Locations
- **来源**：JAMIA (Journal of the American Medical Informatics Association), 2024
- **作者**：Yiming Li, Xueqing Peng等

## 一、解决什么问题

用LLM自动抽取针灸穴位的位置关系，构建结构化穴位知识。

**输入**：穴位描述文本 + 已标注实体
**输出**：关系三元组

## 二、为什么只做RE不做NER

因为实体已有。数据来自WHO Standard Acupuncture Point Locations，361个穴位已整理好。作者之前研究已完成实体抽取，现在只需判断实体间关系。

## 三、数据

- 来源：WHO西太平洋地区针灸穴位定位标准
- 规模：361个穴位（训练288，测试73）
- 实体类型（5类）：Acupoint、Anatomy、Direction、Distance、Subpart
- 关系类型（5类）：direction_of、distance_of、part_of、near_acupoint、located_near

## 四、模型设计

| 模型 | 类型 | 使用方式 |
|---|---|---|
| GPT-3.5 | 闭源 | Zero-shot/Few-shot |
| GPT-3.5 Fine-tuned | 微调 | 专门训练针灸RE |
| GPT-4 | 闭源 | Zero-shot/Few-shot |
| Llama 3-8B | 开源 | Prompt推理 |

## 五、Prompt设计（核心借鉴）

不是让LLM"猜"关系，而是结构化约束：

```
输入：
(1) 原始文本："SP8 is located 3 B-cun inferior to SP9"
(2) 实体列表：T1 Acupoint SP8, T2 Direction inferior, T3 Acupoint SP9
(3) 关系定义：direction_of = 方向关系
(4) 输出格式：R1 direction_of Arg1:T2 Arg2:T3
```

本质：把LLM当成"带医学知识约束的关系分类器"

## 六、实验结果

| 模型 | Micro-F1 |
|---|---|
| GPT-3.5 | 较低 |
| GPT-4 | 较低 |
| Llama 3-8B | 较低 |
| **Fine-tuned GPT-3.5** | **0.92** |

**关键发现**：Fine-tuned GPT-3.5（F1=0.92）击败GPT-4！

**原因**：针灸关系抽取需要的是"领域规则记忆"（低于、旁开、距、位于），不是通用推理能力。领域小任务：专门微调的小模型 > 通用大模型。

## 七、对我们项目的启发

### LLM做中医RE的优劣

| 优势 | 劣势 |
|---|---|
| 少样本能力强 | 稳定性（不同temperature输出不同） |
| 关系扩展容易（改prompt即可） | API成本高 |
| 适合医学文本（规则明确） | 幻觉风险 |

### 比论文17的新发现

- 论文17：传统方案（NER+RE pipeline）
- 本文：从"训练抽取模型"变成"调用推理模型"
- 迁移能力更强，领域适应更容易

### 最佳方案推荐

| 方案 | 推荐 |
|---|---|
| BERT NER+RE | ⭐⭐⭐ |
| RE→NER | ⭐⭐⭐ |
| LLM直接抽三元组 | ⭐⭐⭐ |
| **NER + LLM RE混合** | **⭐⭐⭐⭐⭐** |

**混合架构**：
```
中药文本
  ↓
医学词典/NER模型 → 候选实体
  ↓
LLM关系抽取（给实体列表+关系定义+格式约束）
  ↓
知识图谱
```

NER负责精准找实体，LLM负责灵活判断关系。

## 八、关联笔记

- [[论文07-民族药知识抽取]]
- [[论文12-双粒度嵌套NER]]
- [[论文17-微调LLM构建KG]]
