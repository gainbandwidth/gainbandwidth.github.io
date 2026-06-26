---
title: "MoE 稀疏架构大模型训练与对齐"
date: 2025-12-01
draft: false
tags:
  - LLM
  - MoE
  - PyTorch
  - HuggingFace
categories:
  - 大模型
summary: "构建 209M 参数 MoE 模型，支持 Fine-grained MoE、Shared Expert 和 GQA 结构，在 SFT 与 DPO 对齐任务上取得优异表现。"
ShowToc: true
---

## 项目背景

混合专家（Mixture of Experts, MoE）架构是当前大语言模型提升参数效率的重要方向。本项目旨在从零构建一个具备完整 MoE 架构的稀疏大语言模型，并完成监督微调（SFT）与直接偏好优化（DPO）对齐训练。

## 技术架构

模型参数量 **209M**，采用以下关键设计：

- **Fine-grained MoE**：细粒度专家路由，提升专家利用效率
- **Shared Expert**：共享专家机制，保留通用知识表达能力
- **GQA（Grouped Query Attention）**：分组查询注意力，降低推理时 KV Cache 开销

## 训练流程

1. **预训练**：基于大规模语料完成基础语言能力训练
2. **SFT（监督微调）**：使用约 **200K 条 SFT 样本**进行指令跟随训练
3. **DPO（直接偏好优化）**：通过偏好数据对模型进行人类偏好对齐

## 核心成果

- 在 LLAMA-Factor 评测任务上表现优异
- DPO 对齐后模型输出质量显著提升
- 深入理解了 MoE 路由机制、Expert 负载均衡等关键技术

## 技术栈

`Python` `PyTorch` `HuggingFace Transformers` `PEFT` `DeepSpeed`
