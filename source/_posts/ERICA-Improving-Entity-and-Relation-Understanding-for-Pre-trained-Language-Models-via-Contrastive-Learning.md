---
title: "ERICA: Improving Entity and Relation Understanding for Pre-trained Language Models via Contrastive Learning"
tags:
    - NLP
    - PLM
    - Knowledge Embedding
    - Knowledge Graph
categories: Paper Note
date: 2021-11-27
---
## Motivation

预训练语言模型拥有很强的表征能力，可以利用预训练语言模型生成的语言表征高效捕捉文本的语法和语义特征。作者认为目前平凡的预训练目标不能**明确**建模relational facts，而relational facts对于理解整个文本十分重要。现有的关于建模实体及实体之间关系的研究主要关注于**句内**实体之间的孤立关系，忽略了在整个文档层面上实体之间的关系。

<!--more-->

## Contribution

提出了两个预训练目标，用以更好地捕获in-text facts：

1. Entity Discrimination task(实体区分任务)，用给定的头实体和关系区分出尾实体
2. Relation Discrimination task(关系区分任务)，区分两个给定的关系是否语义上相似

![两个预训练目标](ERICA/1.png)

## Methodology

### 数据构建和形式化描述

ERICA 在大规模无标签的语料上训练，并由外部知识图谱 $\mathcal{K}$ 进行远程监督。

首先将数据以文档为单位分批， $\mathcal{D} = \{d_i\}_{i=1}^{|\mathcal{D}|}$ 是一个包含若干文档的一个 batch。

$\mathcal{E}_i = \{e_{ij}\}_{j=1}^{|\mathcal{E}_i|}$ 是文档 $d_i$ 中出现的所有实体的集合，其中 $e_{ij}$ 是文档 $d_i$ 中的第 $j$ 个实体。

对于文档 $d_i$, 作者枚举出其中的所有实体对 $(e_{ij}, e_{ik})$, 并将它们以 $\mathcal{K}$ 中的关系 $r_{jk}^i$ 连接起来，这样就得到了一个元组集 $\mathcal{T}_i = \{t_{jk}^i=(d_i, e_{ij}, r_{jk}^i, e_{ik})|j\neq k\}$. 对于 $\mathcal{K}$ 中不存在关系的实体对(out-of-KG 问题), 将其关系设置为 `no_relation`. 

对于整个批次，将其中每个文档的元组集拼接得到整个批次的元组集 $\mathcal{T} = \mathcal{T}_i\cup\mathcal{T}_2\cup...\cup\mathcal{T}_{|\mathcal{D}|}$.

进一步，作者通过去除 $\mathcal{T}$ 中所有关系为 `no_relation` 的元素构建了阳性元组集 $\mathcal{T}^+$. $\mathcal{T}^+$ 中包含句内实体对和句间实体对。

### 实体和关系的表征方法

对于每个文档 $d_i$ ，作者首先使用 PLM 对齐进行编码，计算出每个 token 的表征 $\{\mathbf{h}_1, \mathbf{h}_2, ..., \mathbf{h}_{|d_i|}\}$, 然后对于文档中的每个实体 $e_{ij}$, 作者通过对这个实体所包含的所有 token 的表征进行均值池化(*mean pooling*) 获得这个实体本次出现的局部表征。由于一个实体在文档中可能多次出现，其第 $k$ 次出现的局部表征记为：

$$\mathbf{m}_{e_{ij}}^k = \text{MeanPool}(\mathbf{h}_{n_{start}^k}, ..., \mathbf{h}_{n_{end}^k})$$

其中，$n_{start}^k$ 和 $n_{end}^k$ 分别是实体 $e_{ij}$ 所包含的 token 在文档 $d_i$ 中第 $k$ 次出现的起始位置和终止位置。

同时作者将文档中 $e_{ij}$ 的所有局部表征取均值作为其在文档中的全局特征 $\mathbf{e}_{ij}$，**作者认为$\mathbf{e}_{ij}$中包含该实体在该文档中的全部信息**

对于文档中两个实体 $e_{ij_1}, e_{ij_2}$ 之间的关系，作者将其全局表征拼接起来作为其关系的表征 $\mathbf{r}_{j_1j_2}^i = [\mathbf{e}_{ij_1}; \mathbf{e}_{ij_2}]$

### 两个预训练目标

#### Entity Discrimination, ED

实体辨别任务是根据给定的文档和头实体以及关系推断出尾实体。作者认为，这个任务可以促使PLM通过实体之间的关系理解实体。

实践中，先从 $\mathcal{T}^+$ 中采样一个元组 $t_{jk}^{i} = (d_i, e_{ij}, r_{jk}^{i}, e_{ik})$. 然后使用 PLM 区分GT尾实体和文档 $d_i$ 中的其他实体。实际的数据构造成如下的格式：

$$d_i^*=\text{"relation\_name entity\_mention[SEP]}d_i\text{"}$$

即将关系 $r_{jk}^i$ 的名称，头实体 $e_{ij}$ 的 mention(文字) 放在文档 $d_i$ 的前面，并用 $\text{[SEP]}$ 隔开。

该任务的目标就是最大化如下后验概率：

$$\mathcal{P}(e_{ik}|e_{ij}, r_{jk}^i) = \text{softmax}(f(\mathbf{e}_{ik}))$$

其中 $f(\cdot)$ 是一个实体分类器。

作者实验发现直接优化上述后验概率并不能很好地考虑实体之间的关系，因此借鉴对比学习的思想，训练表征使实体正样本$(e_{ij}, e_{ik})$之间的余弦距离比负样本更近。ED任务的损失函数设计为：

$$\mathcal{L}_\text{ED}=-\sum_{t_{jk}^i\in \mathcal{T}^+}\log\frac{\exp(\cos(\mathbf{e}_{ij}, \mathbf{e}_{ik})/\tau)}{\sum_{l=1, l\neq j}^{|\mathcal{E}_i|}\exp(\cos(\mathbf{e}_{ij}, \mathbf{e}_{il})/\tau)}$$

其中的 $\tau$ 是一个超参数。

#### Relation Discrimination, RD

关系辨别任务即辨别两个关系是否语义上相似，与现存的关系增强的PLM相比，本文提出的方法主要是使用了文档级别的关系而不是句内关系的远程监督。作者认为这样可以使PLM学会现实世界中的复杂推理链。作者为实体对之间的关系文本训练表征，并优化使共享相同关系的不同实体对之间的关系表征在语义空间中相近。

实践上，作者从 $\mathcal{T}_s^+$ 或 $\mathcal{T}_c^+$ 中为每种关系线性采样(每种关系对应的采样率与该关系的数量在挡墙batch中的占比成正比)元组对 $t_A=(d_A, e_{A_1}, r_A, e_{A_e}), t_B=(d_B, e_{B_1}, r_B, e_{B_2})$ 其中 $r_A=r_B$. 然后使用**实体和关系的表征方法**中提到的方法对 $r_A$ 和 $r_B$ 进行编码，获得 $\mathbf{r}_{t_A}$ 和 $\mathbf{r}_{t_B}$ . 然后，与ED类似，作者也使用了对比学习的思想构建RD的损失函数：

$$\begin{aligned}
    \mathcal{L}_\text{RD}^{\mathcal{T}_1, \mathcal{T}_2} &= -\sum_{t_A\in \mathcal{T}_1, t_B\in \mathcal{T}_2}\log\frac{\exp(\cos(\mathbf{r}_{t_A}, \mathbf{r}_{r_B})/\tau)}{\mathcal{Z}} \\
    \mathcal{Z} &= \sum_{t_C\in \mathcal{T}/\{t_A\}}^N\exp(\cos(\mathbf{r}_{t_A}, \mathbf{r}_{r_B})/\tau) \\
    \mathcal{L}_\text{RD} &= \mathcal{L}_\text{RD}^{\mathcal{T}_s^+, \mathcal{T}_s^+} + \mathcal{L}_\text{RD}^{\mathcal{T}_s^+, \mathcal{T}_c^+} + \mathcal{L}_\text{RD}^{\mathcal{T}_c^+, \mathcal{T}_s^+} + \mathcal{L}_\text{RD}^{\mathcal{T}_c^+, \mathcal{T}_c^+}
\end{aligned}$$

其中 $N$ 是超参数。实验中确保 $t_B$ 从 $\mathcal{Z}$ 中采样并且从 $\mathcal{T}$ 中构建 $N-1$ 个负样本。

#### 总体训练目标

为避免灾难性遗忘，ERICA 还采用了MLM目标任务，因此总的训练目标是：

$$\mathcal{L} = \mathcal{L}_\text{ED} + \mathcal{L}_\text{RD} + \mathcal{L}_\text{MLM}$$

## 实验

1. 远程监督数据集构建
2. 预训练参数细节
3. RE 任务上与 CNN，BILSTM，BERT，RoBERTa，HINBERT，CorefBERT，SpanBERT，ERINE，MTB，CP比较。分别在 Document-level 和 Sentence-level 上比较
4. Multi-Chioce QA 任务上与 FastQA, BiDAF, BERT, RoBERTa, CorefBERT, SpanBERT, MTB 和 CP 比较。Extractive QA 上与 BERT，RoBERTa，MTB，CP 比较
5. 消融实验，探索了 $\mathcal{L}_\text{ED}$ 和 $\mathcal{L}_\text{RD}$ 的作用，分析了预训练数据的 domain，size 和 实体编码的方法对性能的影响