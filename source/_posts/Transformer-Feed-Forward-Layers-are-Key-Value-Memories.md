---
title: Transformer Feed-Forward Layers are Key-Value Memories
date: 2022-04-03 14:59:27
update: 2022-04-03 14:59:27
authors:
    - Mor Geva
    - Roei Schuster
    - Jonathan Berant
    - Omer Levy
tags:
    - Interpretability
    - FFN
    - Nueral Memory
category:
    - Paper Note
---

# Motivation

Transformer 模型中, FFN 层的参数占 3/2，但是其在网络中的作用还没有很好的被研究和理解

作者提出 FFN 层相当于神经记忆系统，以第一个矩阵为 key， 第二个矩阵为 value 记录了键值对信息。其中的 key 指人类可解释的文本特征(表层的文本结构特征和深层的文本语义特征), value 则可以诱导成在词典空间中的概率分布。

<!--more-->

# Introduction

![How FFN Layers emulate Key-Value Memories](Transformer-Feed-Forward-Layers-Are-Key-Value-Memories/1.png)

FFN 层的计算过程和 Key-value Memories 的计算过程相似： FFN 层中的第一个矩阵是神经键值对记忆的 key 矩阵，第二个矩阵是 value 矩阵。作者在此理论的基础上进行了实验，分析了 FFN 层作为记忆的观点下其究竟记忆了什么信息

实验发现 FFN 中的 key 通常与人类可解释的文本模式相关：即文本结构或语义主题。而 value 可以诱导成词典空间上的概率分布，且该分布与 key 中的文本模式的下一个词有关

作者还发现：Transformer 模型中每一层的 FFN 层都有数百个激活的 memories, 即数百个在词典空间上的概率分布，但是 transformer 层的最终输出的分布与激活的记忆的分布均不相同。这暗示 transformer 层之间的 残差信息 起到主要的预测作用，FFN 层的输出是该残差信息的调整。

## 一些结论

-   FFN 层可以看作未经归一化的 Key-Value Memories
-   FFN 中的 key 可以看作是对输入的模式捕捉：部分 key 向量与输入 x 的乘积较大，说明输入 x 符合该 key 中存储的模式
-   key 中存储的模式是人类可解释的：实验中发现被激活的模式可分为文本结构或语义注意上的模式，均为人类可解释的
-   浅层的 key 主要捕捉浅层的特征：实验中发现浅层的 key 中主要捕捉的是文本表层特征，深层的 key 则主要捕捉语义特征
-   values 可以诱导成词典上的概率分布
-   FFN 层存储的信息主要是关于根据输入直接预测下一词概率分布的
-   每个 记忆单元的概率分布与该层的最终输出通常都不相同，实验发现每层的最终输出通常偏向残差的信息，而不是 FFN 的信息。即使当最终输出与残差不同时(即残差被 FFN 改变), 最终输出也很少会偏向 ffn 的输出。这暗示 ffn 层实际扮演着对 残差信息 的否决者作用，即它智能否决残差中的 top prediction，使之偏向另一个 candidate，而不是偏向 ffn 自己的 top prediction
