---
title: "Dataset and Benchmark"
tags:
    - NLP
    - Dataset
    - Benchmark
categories: Quick Note
---

- SQuAD V1.1: 维基百科上的抽取式问答任务。答案是从给定的文本中抽取的，模型预测出给定的文本中每个token的标记，标记着要抽取的文本段的起始和结尾。
- SQuAD V2.0: 答案未必能够从给定文本中抽取到的问答任务。
- MNLI: 文本二分类任务。预测一个句子是否包含另一个。
- ELI5: 长形式的抽象问答任务。答案由模型基于问题文本和支持材料文本的条件生成，而不是从支持材料中抽取。
- XSum: 高度抽象的新闻总结数据集
- ConvAI2: 对话回应生成任务。基于上下文和角色信息生成回应
- CNN/DM: 新闻总结数据集。总结文本和原始文本密切相关
- RACE: The ReAding Comprehension from Examinations, 大规模阅读理解数据集，来自中国的中学英语考试试题。
- GLUE: The General Language Understanding Evaluation. 是结合9个数据集的benchmark，旨在评估自然语言理解系统。包括单句分类任务和句子对分类任务。