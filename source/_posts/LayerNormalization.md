---
title: "Layer Normalization"
author: 
    - Jimmy Lei Ba
    - Jamie Ryan Kiros
    - Geoffrey E. Hinton 
tags:
    - ML
categories: Paper Note
date: 2021-12-03
---

Layer Normalization 是针对 Batch Normalization 提出的，两者都是深度神经网络中为解决训练困难而提出的归一化手段。

<!--more-->

## Background

前馈神经网络可以认为是一个将输入 $\mathbf{x}$ 映射到输出向量 $y$ 的非线性映射。即下式所示的过程：

$$a_i^{l}={w_i^{l}}^Th^l, h_i^{l+1} = f(a_i^l + b_i^l)$$

其中 $a_i^l$ 指第 $l$ 层第 $i$ 个隐藏单元的表征输出，$w_i^l$ 指第 $l$ 层中第 $i$ 个隐藏单元的权重，$h^l$ 指输入到第 $l$ 层的输入表征，$f(\cdot)$ 是非线性函数，$b_i^l$ 是偏置参数。

这样的深度学习计算过程导致一个层参数的梯度与上一层的输出高度相关，这被称为“协变量偏差”。Batch Normalization 的提出就是为了减弱这种偏差。

### Batch Normalization

BN 将训练样本在每个隐藏单元上的输入归一化，具体地说，对于第 $l$ 层上第 $i$ 个输入，BN 根据训练数据的分布缩放隐藏单元的输入。

$$\overline{a}_i^l = \frac{g_i^l}{\sigma_i^l}(a_i^l - \mu_i^l), \mu_i^l=\mathbb{E}_{\mathbf{x}\sim P(\mathbf{x})}[a_i^l], \sigma_i^l = \sqrt{\mathbb{E}_{\mathbf{x}\sim P(\mathbf{x})}[(a_i^l - \mu_i^l)^2]}$$

其中 $\overline{a}_i^l$ 是第 $l$ 层第 $i$ 个隐藏单元的归一化后的输入，$g_i$ 是增益参数。

上述公式中计算统计参数 $\mu$ 和 $\sigma$ 是在整个训练集 $P(\mathbf{x})$ 上进行，但是这在实践中不可能完成。因为这需要在某个 Batch 的计算中引入不属于该 Batch 的分布信息，因此统计参数由当前的 mini-batch 中的样本中估计。这客观上限制了 mini-batch 的大小不能太小，**并且很难将其应用在 RNN 中**

## Layer Normalization

作者考虑到，如果固定深度神经网络每一层的输入数据的均值和方差，可以减弱上述的协变量偏移问题。LN 中统计参数的计算过程如下：

$$\mu^l = \frac{1}{H}\sum_{i=1}^{H}a_i^l, \sigma^l = \sqrt{\frac{1}{H}\sum_{i=1}^H(a_i^l - \mu^l)^2}$$

其中 $H$ 代表第 $l$ 层(当前层)中隐藏单元的数量。

与 BN 不同的是，LN 中同一层的所有隐藏单元共享相同的统计参数 $\mu$ 和 $\sigma$ ，不同的训练样本计算出不同的 $\mu$ 和 $\sigma$，而 BN 中则是同一 Batch 中所有的训练样本在相同的隐藏单元上共享相同的统计参数，不同的隐藏单元计算出不同的统计参数。这一区别使得 LN 取消了对 mini-batch 尺寸的限制，甚至可以进行 batch-size 为 1 的纯在线训练过程。

## LN 在 RNN 中应用

