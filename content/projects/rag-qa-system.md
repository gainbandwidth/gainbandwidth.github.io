---
title: "垂直领域 RAG 问答系统"
date: 2025-08-01
draft: false
tags:
  - RAG
  - 知识图谱
  - LoRA
  - HuggingFace
categories:
  - 大模型应用
summary: "构建端到端 RAG 流水线，包含检索、重排序和答案生成，通过 LoRA 微调提升垂直领域问答稳定性。"
ShowToc: true
---

## 项目背景

检索增强生成（Retrieval-Augmented Generation, RAG）是解决大模型幻觉问题和引入领域知识的有效手段。本项目针对特定垂直领域，构建了完整的 RAG 问答系统。

## 系统架构

### 检索模块

基于向量数据库实现语义检索，结合 BM25 稀疏检索进行混合检索（Hybrid Retrieval），提升召回率。

### 重排序模块

引入 Cross-Encoder 重排序模型，对检索结果进行精排，过滤低相关性文档片段。

### 答案生成模块

基于检索到的上下文，使用大语言模型生成最终答案，支持引用标注和来源追溯。

## 微调优化

- 使用 **HuggingFace Transformers** 框架进行模型微调
- 采用 **LoRA（Low-Rank Adaptation）** 技术，在保持预训练参数不变的前提下进行低秩适配
- 通过 **Checkpoint** 机制管理训练过程，支持断点续训和版本回滚

## 核心成果

- 端到端 RAG 流水线稳定运行
- LoRA 微调后垂直领域问答准确率显著提升
- 系统支持快速迁移至新领域，只需替换知识库和少量微调数据

## 技术栈

`Python` `HuggingFace Transformers` `PEFT/LoRA` `LangChain` `向量数据库`
