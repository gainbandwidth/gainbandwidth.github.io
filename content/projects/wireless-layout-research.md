---
title: "无线知识图谱生成与跨域拓扑优化"
date: 2026-05-01
weight: 1
draft: false
tags:
  - 知识图谱
  - GraphSAGE
  - 强化学习
  - 因果发现
  - 论文
  - 研究
categories:
  - 科研
summary: "学位论文研究，提出 Causal-RL-GNN 框架，融合因果搜索空间、跨域迁移与强化学习优化，实现无线知识图谱的自动拓扑生成。论文已被 VTC 2026 Fall (Boston) 录用。"
ShowToc: true
---

## 论文信息

**Causal-Enhanced Graph Reinforcement Learning for Cross-Domain Topology Optimization**

作者：Bowen Gu, Peng Ren, Hang Zhan, Yongming Huang

单位：东南大学、紫金山实验室

发表于：**VTC 2026 Fall — Boston, MA, USA**

基金项目：国家自然科学基金（Grant No. 62225107）

[📄 下载论文 PDF](/papers/VTC2026_Gu_et_al.pdf)

## 研究背景

无线知识图谱（Wireless Knowledge Graph, WKG）以图结构建模通信网络中实体（基站、用户、信道等）间的复杂关系，为网络拓扑优化提供了统一的表示框架。然而，现有方法面临两大挑战：一是如何从海量 KPI 时间序列中准确发现因果关系以构建合理的图拓扑；二是当目标域数据稀缺时，如何利用源域知识实现有效的跨域迁移。

本研究提出 **Causal-RL-GNN** 框架，通过因果发现、跨域迁移和强化学习三个模块的协同工作，实现无线知识图谱的自动生成与优化。

## 整体框架

![Causal-RL-GNN 框架流程图](/images/flowchart.png)

框架包含三个核心组件（对应图中 A、B、C 三部分）：

### A. 因果搜索空间构建（Causal Search Space）

从 5G RAN 的 KPI 时间序列出发，计算节点间的**时滞相关（Lagged Correlation）**，通过阈值筛选生成候选边集合。这一过程将无结构的 KPI 数据转化为具有因果意义的候选拓扑集合，为后续强化学习提供结构化的动作空间。

### B. 跨域迁移（Cross-Domain Transfer）

采用**预训练 Critic 网络**实现知识迁移。在数据充足的源域上预训练 Critic，使其学习拓扑质量的评估能力；然后通过微调（Fine-tuning）将这一能力迁移到数据稀缺的目标域，显著降低目标域的训练成本。

### C. 强化学习优化循环（RL Optimization Loop）

Actor 网络基于 **GraphSAGE** 编码当前图状态并输出拓扑修改动作（添加/删除边），Critic 网络基于 **GIN（Graph Isomorphism Network）** 评估动作价值。两者在真实网络环境的奖励信号下交替训练，逐步优化生成策略。

## 实验验证

在 5G RAN KPI 数据集上进行实验，结果表明：

- 因果搜索空间有效缩小了动作空间，提升收敛速度
- 跨域迁移在目标域数据有限时仍保持良好的拓扑生成质量
- Causal-RL-GNN 在拓扑优化指标上优于基线方法

## 技术栈

`Python` `PyTorch Geometric` `GraphSAGE` `GIN` `强化学习` `因果发现` `5G RAN`
