---
title: Integrating Multimodal Information in Large Pretrained Transformers
date: 2022-03-23 17:20:49
update: 2022-03-23 17:20:49
tags: 
    - Multimodal
    - Visual Knowledge
    - Aoustic Knowledge
categories:
    - Paper Note
---

作者基于一种观点：结合多模态信息的语言表征与不结合多模态信息的语言表征在表征空间中的位置不同，因此，可以将多模态信息视为表征空间中的 **移位向量**，与不结合多模态信息的语言表征向量相加可以得到向量空间中的最终位置。

<!--more-->

# Multimodal Adaptation Gate(MAG)

作者基于上述观点，设计了多模态融合架构 MAG 结合 BERT 和 XLNet 在情感分类任务上进行了验证。

MAG 试图将视觉信息和音频信息融入语言表征。数据集中，作者首先使用机器翻译生成视频字幕，然后人工修正，作为语言信息。随后作者根据时间共现性提取出 <图像，音频，字幕> 的序列，其中每个三元组是同时出现的。

MAG 随后提取图像中的特征信息(任务的表情，手部动作等)和语音中的特征信息(语调等)，并编码为向量与语言信息融合。

记序列中第 $i$ 个元组的语言嵌入向量，语音嵌入向量和视觉嵌入向量为 $(Z_i, A_i, V_i)$, 作者首先将语言向量分别和其他模态向量拼接，分别计算**出门控向量** $g_i^v$ 和 $g_i^a$:

$$\begin{aligned}
    g_i^v &= R(W_{gv}[Z_i; V_i] + b_v)\\
    g_i^a &= R(W_{ga}[Z_i; A_i] + b_a)    
\end{aligned}$$

其中 $R$ 是激活函数. 作者认为，上述门控网络可以计算出多模态信息与语言信息的相关系数。作者随后使用上述门控系数与多模态嵌入向量相乘获得多模态信息融合向量，即：表征空间中的位移向量：

$$H_i = g_i^a\cdot(W_aA_i) + g_i^v\cdot(W_vV_i) + b_H$$

随后，按照位移向量的观点，作者将多模态信息向量和语言向量相加获得**多模态信息下的语言表征**：

$$\bar{Z}_i = Z_i + \alpha H_i$$

其中 $\alpha = \min(\frac{\|Z_i\|_2}{\|H_i\|_2}\beta, 1)$

![MAG](Integrating-Multimodal-Information-in-Large-Pretrained-transformers/1.png)

上述 MAG 模型将多模态信息向量与语言表征向量做了融合，作者随后介绍了 MAG 与 BERT 的结合。

## MAG-BERT

作者选择将 MAG 模型夹在 BERT 的某两层之间，不妨记作第 $j$ 层和第 $j + 1$ 层之间，则 MAG 的语言表征来自 BERT 第 $j$ 层 transformer layer 的输出，且自 $j + 1$ 层起，输入向量中融入了多模态信息：

![MAG-BERT](Integrating-Multimodal-Information-in-Large-Pretrained-transformers/2.png)

在情感分类任务中，作者直接取序列开头的 $[CLS]$ token 的表征作为整个序列的表征。

## MAG-XLNet

MAG-XLNet 的结构与 MAG-BERT 相同，不同之处来自 XLNet 本身：在 XLNet 中，$[CLS]$ token 加在序列的尾部。

## experiment

作者在 CMU-MOSI 数据集上进行了实验，该数据集是多模态情感分类数据集，数据来自 youtube 电影评论视频。

除此之外，作者试验了 MAG 可以放置的位置：从嵌入层之后第一层 transformers 之前到最后一层 transformer 之前，结果显示放在第一层 transformer 之后性能最好。这一结果是符合直觉的：嵌入层的语言表征没有上下文信息，而在太靠后的层加入多模态信息则无法对其进行有效的上下文融合。