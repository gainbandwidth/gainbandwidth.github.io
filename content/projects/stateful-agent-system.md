---
title: "MewCode —— 终端 AI 编码助手"
date: 2026-03-01
weight: 3
draft: false
tags:
  - Agent
  - LLM
  - MCP
  - Python
  - Textual
  - MewCode
categories:
  - AI Agent
summary: "轻量级终端 Coding Agent，基于 ReAct 与 Plan Mode 双模式驱动 LLM 自主完成编程任务，五层分层架构，支持多模型、MCP、Skill、跨会话记忆、多 Agent 协作。"
ShowToc: true
---

## 项目概览

**技术栈**：Python, MCP, ReAct, Skill, Multi-Agent

轻量级终端 Coding Agent，基于 ReAct 与 Plan Mode 双模式驱动 LLM 自主完成编程任务。交互、引擎、工具、记忆、安全五层分层架构，支持 Anthropic、OpenAI 双协议、MCP 工具扩展、Skill 技能包、跨会话记忆、多 Agent 并行协作。

## 五层架构

```
┌─────────────────────────────────────────────┐
│  交互层：CLI (Textual TUI)、Slash Command、Skill │
├─────────────────────────────────────────────┤
│  引擎层：对话模块、Agent 循环、System Prompt      │
├─────────────────────────────────────────────┤
│  工具层：25+ 内置工具、MCP 协议、Hook 生命周期     │
├─────────────────────────────────────────────┤
│  记忆层：上下文压缩、会话持久化、自动记忆提取       │
├─────────────────────────────────────────────┤
│  安全层：权限拦截、路径沙箱、Human-in-the-Loop     │
└─────────────────────────────────────────────┘
```

## 个人职责

- 独立负责五层分层架构设计与全部核心技术决策，借助 Claude Code 完成模块编码与调试
- 设计并实现跨会话记忆系统，包括 JSONL 持久化、LLM 自动记忆提取与项目指令文件分级加载
- 负责核心技术方案选型，包括多 LLM 协议适配、上下文两层压缩策略、MCP 工具延迟加载与 Skill 技能体系方案

## 技术亮点

### MCP 工具延迟加载
MCP 工具注册时只传名称不传完整 schema，LLM 按需通过 ToolSearch 发现后再加载完整定义，**百级工具场景下工具描述 Token 占用减少 85%**，避免上下文窗口被工具定义挤占。

### 多 LLM 协议适配
统一 Anthropic、OpenAI 两套不同的流式响应协议为同一套内部事件接口（StreamEvent），**新增 LLM 供应商只需适配一个接口**。

### 两层上下文压缩
设计两层渐进式上下文压缩策略：Layer 1 每轮无条件落盘裁剪零 LLM 调用，Layer 2 在 80% 阈值时触发全量摘要，自动对齐 Function Calling 调用配对约束，**支持数小时连续编程会话不丢失关键上下文**。

### 五层权限拦截
覆盖命令拦截、路径沙箱、规则引擎、权限模式、人工确认的五层纵深权限模型，**任一层拒绝即终止操作，支持 Agent 全自动模式安全执行**。

### 自动记忆提取
每轮对话结束后异步调 LLM 回顾对话内容，自动将用户偏好、纠正反馈、项目知识、参考信息四类记忆分类持久化到独立文件，**跨会话知识持续累积，同类错误纠正一次后自动规避，新会话自动继承项目上下文无需重复交代**。

### 多 Agent 协作
将复杂任务拆分给多个 Agent 并行处理，基于 Git Worktree 实现文件级隔离避免编辑冲突，Coordinator Agent 只负责任务拆分与结果汇总不参与编码，**突破单 Agent 上下文窗口瓶颈，大型任务处理效率成倍提升**。

## 子模块详解

本项目按模块拆解为以下系列文章，分为**骨架篇**（核心引擎）和**进阶篇**（扩展能力）两部分。

### 骨架篇：核心引擎七大模块

| # | 模块 | 文章 |
|---|------|------|
| 1 | 开口说话 | [让 LLM 开口说话：对话模块与多协议适配](/posts/2026/06/mewcode-01-first-words/) |
| 2 | 工具系统 | [工具系统：25+ 内置工具与并发分批执行](/posts/2026/06/mewcode-02-tool-system/) |
| 3 | Agent 循环 | [Agent 循环：ReAct 与 Plan Mode 双模式驱动](/posts/2026/06/mewcode-03-agent-loop/) |
| 4 | System Prompt | [System Prompt 设计：优先级排序的 PromptBuilder](/posts/2026/06/mewcode-04-system-prompt/) |
| 5 | 权限系统 | [五层权限拦截：从命令检测到 Human-in-the-Loop](/posts/2026/06/mewcode-05-permissions/) |
| 6 | MCP | [MCP 协议集成：延迟加载与工具生态扩展](/posts/2026/06/mewcode-06-mcp/) |
| 7 | 上下文管理 | [两层上下文压缩：数小时会话不丢失](/posts/2026/06/mewcode-07-context/) |

### 进阶篇：从单 Agent 到多 Agent

| # | 模块 | 文章 |
|---|------|------|
| 8 | 记忆系统 | [跨会话记忆：自动提取、JSONL 持久化与 Recall 预取](/posts/2026/06/mewcode-08-memory/) |
| 9 | Slash Command | [Slash Command：斜杠命令系统与交互设计](/posts/2026/06/mewcode-09-slash-command/) |
| 10 | Skill | [Skill 技能包：Markdown 驱动的标准化操作流程](/posts/2026/06/mewcode-10-skill/) |
| 11 | Hook | [Hook 系统：10 个生命周期事件的 Shell 钩子](/posts/2026/06/mewcode-11-hook/) |
| 12 | SubAgent | [SubAgent：子智能体派生与异步执行](/posts/2026/06/mewcode-12-subagent/) |
| 13 | Worktree | [Git Worktree 隔离：并行 Agent 的文件级安全](/posts/2026/06/mewcode-13-worktree/) |
| 14 | Agent Teams | [Agent Teams：团队协作与 Coordinator 模式](/posts/2026/06/mewcode-14-teams/) |

## 技术栈

`Python 3.11+` `Textual` `Anthropic SDK` `OpenAI SDK` `MCP` `asyncio` `Pydantic` `Hatchling`
