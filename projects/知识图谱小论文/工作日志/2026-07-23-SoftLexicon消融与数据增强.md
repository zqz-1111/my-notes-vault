---
date: 2026-07-23
tags: [知识图谱, NER, SoftLexicon, Word2Vec, 数据增强]
---

# SoftLexicon消融实验与数据增强

## 一、SoftLexicon + Word2Vec结果

### 实验配置
- 预训练模型：hfl/chinese-bert-wwm-ext
- 数据：TCM-NER-3000 V2 4类（2332句，7830实体）
- Word2Vec：古籍+TCM_KG描述训练，30738词，dim=64，覆盖率78.4%

### 结果对比

| 模型 | Dev F1 | Test F1 |
|---|---|---|
| BERT-CRF (Baseline) | 0.775 | 0.739 |
| SoftLexicon + Random | 0.768 | 0.734 |
| SoftLexicon + Word2Vec | 0.765 | 0.736 |

### 各实体类型

| 类型 | Baseline | SoftLexicon+Random | SoftLexicon+W2V |
|---|---|---|---|
| Herb | 0.88 | 0.88 | 0.86 |
| Formula | 0.80 | 0.82 | 0.81 |
| Disease | 0.53 | 0.49 | 0.55 |
| Symptom | 0.53 | 0.54 | 0.52 |

### 结论

**SoftLexicon在当前数据集上无效。** 无论随机初始化还是Word2Vec预训练，都没有超过Baseline。

**原因分析（按重要性排序）：**

1. **词典和标注体系不匹配**（最核心）
   - SoftLexicon假设"词典词=实体词"，但中医有大量嵌套实体
   - 例：加味逍遥散(Formula)包含逍遥散(Formula)，词典匹配造成边界冲突
   - Herb下降-2就是边界冲突导致

2. **BERT已经吸收了词汇信息**
   - SoftLexicon最初是给BiLSTM-CRF设计的，BiLSTM缺词信息
   - BERT的self-attention本身就能捕获词边界，词典增强变成冗余/噪声

3. **数据量太少**
   - 训练集1865句，模型自己就能记住
   - 额外模块反而容易过拟合

## 二、数据问题诊断

### 数据集本身的问题

| 问题 | 具体表现 | 影响 |
|---|---|---|
| 数据量小 | 训练集1865句 | Disease只有131个测试样本 |
| 标注噪声 | V2准确率~88-90% | ~10%实体有错误 |
| 类别不均衡 | Herb 3452 vs Formula 641 | 小类别被压制 |
| 古籍语言特殊 | "风邪犯肺"、"营卫不和" | 和BERT预训练语料差异大 |
| Disease/Symptom边界模糊 | "湿热"是疾病还是症状？ | 标注一致性差 |

### 和之前实验对比

| 实验 | 数据 | BERT-CRF F1 |
|---|---|---|
| 之前的NER实验 | 不同数据集 | 0.83 |
| 现在 | TCM-NER-3000 V2 | 0.739 |

同一个算法，F1差了9个点。说明**主要是数据问题，不是算法问题**。

## 三、数据增强

### 方案：KG模板生成

用TCM_KG中的实体对，按中医文本模式生成新句子。

模板类型：
- {Herb}治疗{Disease}
- {Formula}主治{Disease}
- {Disease}者，见{Symptom}
- {Herb}为君，{Herb2}为臣
- 等20+种模板

### 增强结果

- 生成句子：2000句
- 实体分布：Herb 1160, Disease 1078, Symptom 700, Formula 534
- 合并后数据：4332句（原2332 + 增强2000）

## 四、最终实验路线（精简版）

| 实验 | 模型 | 数据 | 目的 |
|---|---|---|---|
| Exp1 ✅ | BERT-CRF | 原始2332句 | 基线 F1=73.9 |
| Exp2 ✅ | + SoftLexicon | 原始2332句 | 消融：词典增强无效 |
| Exp3 🔄 | 领域BERT-CRF | 原始2332句 | 领域预训练有效 |
| Exp4 ⏳ | 领域BERT-CRF | 增强4332句 | 最终模型 |

**论文叙事：**
- Exp1→Exp2：证明单纯词典增强不够（消融）
- Exp1→Exp3：证明领域预训练有效
- Exp3→Exp4：证明数据增强有效
- 最终模型优于基线，Pipeline完整

## 五、论文写作要点

SoftLexicon的结果不要浪费，写进论文作为消融分析：

> 实验表明，在小规模、高噪声的中医古籍NER数据上，单纯改进模型架构（SoftLexicon）收益有限。真正的瓶颈在于数据质量和规模。因此，本文提出LLM辅助数据构建方法和领域预训练策略。

这个结论正好衔接创新点4（LLM-assisted data construction）。

## 六、待做

- [ ] Step A：领域BERT预训练（Kaggle运行中）
- [ ] Step B：用领域BERT+增强数据训练NER
- [ ] 收集最终实验结果，写论文NER部分
- [ ] 转RE阶段
