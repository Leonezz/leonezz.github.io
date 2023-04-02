---
title: "Improving Multi-hop Question Answering over Knowledge Graphs using Knowledge Base Embeddings"
author: 
    - Apoorv Saxena
    - Aditay Tripathi
    - Partha Talukdar
tags:
    - Knowledge Graph
    - QA
    - Embedding
categories: Paper Note
---

Multi-hop KGQA 要求跨越 Knowledge Graph, KG 的多个边进行推理，而且 KG 往往是不完整的，使 KGQA 更具挑战性。最近的研究有些使用额外的相关领域的语料解决 KG 的稀疏性问题，但是额外的语料是否可获取，以及*相关性*如何判别都是问题。KG embedding 作为另一个研究方向，被用于缺失连接预测以及 KG 补全，但是没有被直接用于 Multi-hop KGQA 问题。本文作者提出 EmbedKGQA，使用 KG embedding 以及 Question embedding 解决 KGQA 问题。

<!--more-->

## Introduction

对于 multi-hop KGQA 问题，主要挑战在于：

- 系统需要在 KG 的多条边上进行推理
- KG 往往是**不完整**的

为解决 KG 的稀疏性，Sun et al.[^pullnet][^Sun2018] 利用额外的相关领域文本语料来应对。其中 Sun et al. 提出的 PullNet[^pullnet] 从 KG 中构建一个与问题相关的子图，然后利用额外的文档对其进行增强，随后在增强后的子图上使用图卷积神经网络计算最终答案。这种方法不仅存在*相关的额外的文本语料*的获取问题，同时也需要一个预定义的邻域范围用于子图的召回。

另一个方向上，Bordes et al.[^transe], Trouillon et al.[^Trouillon2016], Yang et al.[^Yang2014a] 和 Nickel et al.[^Nickel2011] 使用 KG embedding 的方法为 KG 中的**实体**和**关系**学习高维嵌入，然后将其用于连接预测以及图谱补全。

## Related Work

- **KGQA**:
  - TransE[^transe] 等用于单跳场景
  - Yih et al.[^Yih2015] 和 Bao et al., 2016[^Bao2016] 等提出从 KG 中抽取一个特定于问题的子图来回答问题
  - Bordes et al., 2014a[^Bordes2014a] 提出的方法将抽取出的关于头实体的子图编码到高维空间用于QA任务
  - Bordes et al., 2015[^memnet] 提出使用 Memory Network 为知识图谱中出现的事实训练表征用于QA
  - Bordes et al., 2014b[^Bordes2014b] 训练一个问题与三元组的相似度函数，然后在测试时为问题和候选三元组计算相似度分数
  - Yang et al., 2014b 和 Yang et al., 2015 利用嵌入的方法将自然语言问题转换为逻辑形式
- **KG embedding**: 
  - Bordes et al. 提出的 TransE[^transe]，将知识图谱嵌入到高维实空间，将关系看作是头实体和尾实体之间的 **translation**
  - Sun et al. 提出的 RotatE[^rotate]，将知识图谱中的实体嵌入到复空间，并将关系看作是复空间之间的 **rotate**

## Background

### KG

给定实体集合 $\mathcal{E}$ 和关系集合 $\mathcal{R}$, 知识图谱 $\mathcal{G}$ 定义为三元组的集合 $\mathcal{K}\subseteq\mathcal{E}\times\mathcal{R}\times\mathcal{E}$. 其中每个三元组记作 $(h, r, t)$, $h, t$ 分别代表头实体和尾实体，$r$ 代表它们之间的关系。

### Link Prediction

给定一个**不完整**的 KG, Link Predition 的任务是预测缺失的连接。KG Embedding 模型通过一个评分函数 $\phi$ 为三元组评分 $s=\phi(h, r, t)$ 指示该连接是否存在。

### KG Embedding

对于 $e\in\mathcal{E}$ 和 $r\in\mathcal{R}$, KG Embedding 为其生成一个嵌入向量 $e_e\in \mathbb{R}^{d_e}, e_r\in\mathbb{R}^{d_r}$.
嵌入通常通过对比学习的方法训练，即采用一个 $(e_h, e_r, e_t)$ 的评分函数 $\phi$, 训练模型为三元组 $(h, r, t)$ 生成嵌入。使得对于 $(h, r, t) \in \mathcal{K}$, $\phi(e_h, e_r, e_t) > 0$; 对于 $(h', r', t') \notin \mathcal{K}$, $\phi(e_{h'}, e_{r'}, e_{t'}) < 0$.

### ComplEx Embedding

ComplEx[^Trouillon2016] 是一种通过张量分解来嵌入实体和关系到**复空间**的方法。对于 $h, t\in\mathcal{E}$ 和 $r\in \mathcal{R}$, ComplEx 生成嵌入 $e_h, h_r, h_t \in \mathbb{C}^d$, 并且定义评分函数为：
$\begin{aligned}
  \phi(h, r, t) &= \text{Re}(\langle e_h, e_r, e_t\rangle)\\
  &=\text{Re}(\sum_{k=1}^{d}e_h^{(k)}e_r^{(k)}e_t^{(k)})
\end{aligned}$
  
使得对于三元组正例评分大于0，负例评分小于0.

## EmbedKGQA

KGQA: 给定一个自然语言问题 $q$ 和一个问题中出现的主题实体 $e_h$, KGQA 的任务是从 KG 中抽取出问题 $q$ 指向的那个实体 $e_t$

作者提出的 EmbedKGQA 模型包含三个模块：
1. KG Embedding 模块，为 KG 中所有实体创建 embedding
2. Question Embedding 模块，为自然语言问题创建 embedding
3. Answer Selection 模块，缩小备选答案的范围并选择最终答案

### KG Embedding 模块

KG Embedding 直接使用 ComplEx Embedding 方法，根据训练集中实体对 KG 的覆盖程度，训练出的 embedding 可以选择保持不变或者在后续步骤中微调。

### Question Embedding 模块

该模块将自然语言问题 $q$ 嵌入到复向量空间 $\mathbb{C}^d$. 这个过程首先使用一个预训练的 RoBERTa 将 $q$ 编码为 768 维的向量，然后使其通过四层带有 ReLU 激活函数的线性全连接层，最终投影到 $\mathbb{C}^d$.

Question Embedding 的训练过程同样采用 ComplEx Embedding，对于自然语言问题 $q$，主题实体 $h\in\mathcal{E}$ 和答案集合 $\mathcal{A}\subseteq\mathcal{E}$, QE 的训练即使得：

$\begin{aligned}
  \phi(e_h, e_q, e_a) > 0 &\ \ \forall a \in \mathcal{A}\\
  \phi(e_h, e_q, e_{\overline{a}}) < 0 &\ \ \forall \overline{a} \notin \mathcal{A}
\end{aligned}$

### Answer Selection 模块

在推理时，模型为 $(h, q)$ 对与所有的候选答案 $a'\in \mathcal{E}$ 组成的三元组评分。对于相对较小的 KG，直接选出评分最高的实体作为答案：

$$e_{ans} = \argmax_{a'\in\mathcal{E}}\phi(e_h, e_q, e_{a'})$$

但是当 KG 很大时，适当修剪候选答案集合将会显著提升性能。

#### Relation Matching

与 PullNet[^pullnet] 类似，作者通过一个评分函数 $S(r, q)$ 来根据自然语言问题 $q$ 排序关系 $r\in \mathcal{R}$. 记 $h_r$ 为关系 $r$ 的嵌入，$q'=(<s>, w_1, ..., w_{|q|}, </s>)$ 为问题 $q$ 输入 RoBERTa 的序列，则评分函数定义为 RoBERTa 的最终隐藏层输出和 $h_r$ 的点积再经过 sigmoid 函数：

$\begin{aligned}
  h_q = \text{RoBERTa}(q')\\
  S(r, q) = \text{sigmoid}(h_q^Th_r)
\end{aligned}$

在所有关系中，选择使得上述评分函数大于0.5的部分记作 $\mathcal{R}_a$, 对于每个候选答案实体 $a'\in \mathcal{E}$，找到从 $h$ 到 $a'$ 的最短路径，这个路径由关系构成，将路径上的关系记作 $\mathcal{R}_{a'}$, 则候选实体 $a'$ 的 relation score 定义为 $\mathcal{R}_a$ 与 $\mathcal{R}_{a'}$ 的交集大小：$\text{RelScore}_{a'} = |\mathcal{R}_a\cap\mathcal{R}_{a'}|$

最终使用 relation score 和 ComplEx score 的线性组合寻找答案：

$$e_{ans} = \argmax_{a'\in\mathcal{N}_h}\phi(e_h, e_q, e_{a'}) + \gamma*\text{RelScore}_{a'}$$



[^pullnet]: **Pullnet: Open domain question answering with iterative retrieval on Knowledge bases and text**. *Haitian Sun, Tania Bedrax-Weiss, William W Cohen*. [arXiv: 1904.09537](https://arxiv.org/abs/1904.09537)
[^Sun2018]: **Open domain question answering using early fusion of knowledge bases and text**. *Haitian Sun, Bhuwan Dhingra, Manzil Zaheer, Kathryn Mazaitis, Ruslan Salakhutdinov, William W Cohen*. [arXiv: 1809.00782](https://arxiv.org/abs/1809.00782)
[^transe]: **Translating embeddings for modeling multi-relational data**. *Antoine Bordes, Nicolas Usunier, Alberto Garcia-Duran, Jason Weston, Oksana Yakhnenko*. [NIPS 2013](https://proceedings.neurips.cc/paper/2013/hash/1cecc7a77928ca8133fa24680a88d2f9-Abstract.html)
[^Trouillon2016]: **Complex embeddings for simple link prediction**. *Theo Trouillon, Johannes Welbl, Sebastian Riedel, Eric Gaussier, Guillaume Bouchard*. [ICML 2016](http://proceedings.mlr.press/v48/trouillon16.html)
[^Yang2014a]: **Embedding entities and relations for learning and inference in knowledge bases**. *Bishan Yang, Wen-tau Yih, Xiaodong He, Jianfeng Gao, Li Deng*. [arXiv: 1412.6575](https://arxiv.org/abs/1412.6575)
[^Nickel2011]: **A three-way model for collective learning on multi-relational data**. *Maximilian Nickel, Volker Tresp, Hans-Peter Kriegel*. [ICML 2011](https://icml.cc/2011/papers/438_icmlpaper.pdf)
[^rotate]: **Rotate: Knowledge graph embedding by relational rotation in complex space**. *Zhiqing Sun, Zhi-Hong Deng, Jian-Yun Nie, Jian Tang*. [arXiv: 1902.10197](https://arxiv.org/abs/1902.10197)
[^Yih2015]: **Semantic parsing via staged query graph generation: Question answering with knowledge base**. *Scott Wen-tau Yih, Ming-Wei Chang, Xiaodong He, Jianfeng Gao*. [ACL 2015](https://aclanthology.org/P15-1128.pdf)
[^Bao2016]: **Constraint-based question answering with knowledge graph**. *Junwei Bao, Nan Duan, Zhao Yan, Ming Zhou, Tiejun Zhao*. [COLING 2016](https://aclanthology.org/C16-1236.pdf)
[^Bordes2014a]: **Question answering with subgraph embeddings**. *Antoine Bordes, Sumit Chopra, Jason Weston*. [arXiv: 1406.3676](https://arxiv.org/abs/1406.3676)
[^memnet]: **Large-scale simple question answering with memory networks**. *Antoine Bordes, Nicolas Usunier, Sumit Chopra, Jason Weston*. [arXiv: 1506.02075](https://arxiv.org/abs/1506.02075)
[^Bordes2014b]: **Open question answering with weakly supervised embedding models**. *Antoine Bordes, Jason Weston, Nicolas Usunier*. [arXiv: 1404.4326](https://arxiv.org/abs/1404.4326)