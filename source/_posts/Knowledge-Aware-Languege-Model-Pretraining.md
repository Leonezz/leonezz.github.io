---
title: Knowledge-Aware Languege Model Pretraining
date: 2022-03-12 16:44:33
update: 2022-03-12 16:44:33
authors:
    - Corby Rosset
    - Chenyan Xiong
    - Minh Phan
    - Xia Song
    - Paul Bennett
    - Saurabh Tiwary
tags:
    - Knowledge Injection
    - Entity Signal 
categories: Paper Note
---

文章研究向 PLM 注入知识的方法，提出一种不需更改 transformer layer，不需添加额外的 knowledge-aware layer 的知识注入方法：KALM。

<!--more-->

## Introduction

PLM 生成的语言表征被认为含有丰富的词法语法知识，但是在缺乏显示知识性任务训练时，PLM 常常生成语法上正确，但事实上错误的文本。这说明 PLM 中缺乏语义，以及事实知识。

作者提出一种促使 PLM 注意输入文本中的实体以及其在文本中的角色的预训练模式，在没有使模型变得更大的情况下向 PLM 注入知识。

作者首先将输入语句中的 word span 连接到其指代的实体上，然后同时为 word 生成 word embedding 和 entity embedding。在输出层，除去 PLM 的语言建模目标之外，作者添加了一个 entity prediction task 引导模型从干扰项中分辨出 word 所指向的实体。这两个训练目标综合起来即显示地引导模型不仅要预测出正确的词（语法，句法，语义知识），还要预测出这些词所指代的实体（事实知识）。

## KALM

作者在自回归模型的基础上设计 KALM，对于一个 $n$ 个 tokens 的序列 $X = \{w_1, w_2, ..., w_n\}$, 自回归模型由以下语言概率分解描述：

$$p(X) = \prod_i p(w_i|w_{<i})$$

在 PLM 中，通常由 transformer layer 计算上述概率： $p(w_i|w_{<i}) = \text{transformer}(w_i|w_{<i})$

上述过程中，PLM 间接通过词之间的共现模式捕捉语义知识。作者则通过向模型提供一个**信号**提示输入/输出中**实体**的存在以促使模型对**知识**的注意，期望 PLM 能够从输入语句中捕捉**事实知识**

### Entity Tokenizer

作者首先使用一个 Entity Tokenizer 将输入文本中所有的 token 与其所指代的最常出现的实体连接：$w_{i:i+k}\rightarrow e_i$, 其中 $e_i$ 是 word span $w_{i:i+k}$ 所最经常指代的实体。当 $w_i$ 不属于任何已知的实体时：$e_i = null$. 上述过程通过在一个预定义的实体词典上进行文本匹配进行。

经过 tokenize 后，输入语句被分成词-实体两个 token 序列：

$$X_{duet} = \begin{cases}
    \{w_1, w_2, \dots, w_T\} &\text{Word Sequence}\\
    \{e_1, e_2, \dots, e_T\} &\text{Entity Sequence}
\end{cases}$$

上述两个序列逐位置对齐，当有多个词对应同一个实体时，如 $w_{i:i+k}$ 对应一个实体，则 $e_i$ 到 $e_{i+k}$ 是相同的。

### Knowledge-Aware Input

经过 tokenize 后，作者为两个 token 序列分别生成嵌入：

$$\begin{aligned}
    \mathbf{e}_i &= \text{Embedding}_e(e_i)\in\mathbb{R}^{d_e}\\
    \mathbf{w}_i &= \text{Embedding}_w(w_i)\in\mathbb{R}^{d_w}
\end{aligned}$$

两个嵌入线性相加作为模型的输入嵌入：$\mathbf{t}_i = \mathbf{w}_i + \text{Linear}_t(\mathbf{e}_i)$, 其中 $\text{Linear}_t\in\mathbb{R}^{d_e\times d_w}$

### Knowledge-Aware Output

在输出层上，除自回归模型的 next-word prediction 任务外，作者添加了一个 next-entity prediction 任务。

具体来说，作者添加了一个 output head 进行实体辨别。记 $L$ 层 transformer 层输出的第 $i$ 个 token 的表征为 $\mathbf{h}_i^L$, 则第 $i$ 个位置的实体损失计算为：

$$\begin{aligned}
    l_e(e_i|t_{<i}) &= \max(0, s(\mathbf{h}_i^L, \mathbf{e}_i) - s(\mathbf{h}_i^L, \mathbf{e}_-)+\lambda)\\
    s(\mathbf{h}_i^L, \mathbf{e}_j) &= \cos(\text{Linear}(\mathbf{h}_i^L), \mathbf{e}_j)\\
    \mathbf{h}_i^L &= \text{transformer}^L(t_{<i})
\end{aligned}$$

上式中，$e_i$ 指第 $i$ 个 token 所指代的实体，$e_-$ 指作者从除 $e_i$ 之外的实体中采样得到的负例，该损失促使模型分辨 $token_i$ 所指代的实体。

值得注意的是，在实验中，作者使用的负例采样策略为：1% 的 $null$ , 49% 的随机采样实体，50% 的从目标实体的 Trans-E  嵌入空间中最近的 100 个实体中采样（被认为是难以分辨的负例）。

### Pretraining

KALM 的总体损失是：

$$l_\text{KALM}(X_{duet} = \sum_i l_w(p(w_i|t_{<i})) + \alpha l_e(e_i|t_{<i})$$

### Inference

推理时，仅使用 word prediction head。

## 实验

作者进行了知识嗅探评测和 zero-shot QA 任务评测。