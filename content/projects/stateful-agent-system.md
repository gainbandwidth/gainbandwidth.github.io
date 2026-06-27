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
categories:
  - AI Agent
summary: "基于 LLM-in-a-loop 架构的终端编码助手，五层架构设计，支持 25+ 内置工具、MCP 协议、多智能体协作、上下文压缩与长期记忆。"
ShowToc: true
---

## 项目简介

MewCode 是一个**终端 AI 编码助手**，采用 LLM-in-a-loop 架构：模型自主推理任务、调用工具、处理结果并迭代循环，直到任务完成。项目对标 Claude Code，从零用 Python 实现了完整的编码智能体系统。

## 五层架构设计

系统采用严格的五层分层架构，各层职责清晰：

### 交互层（Interaction）

基于 [Textual](https://github.com/Textualize/textual) 构建终端 TUI 界面，支持斜杠命令系统、技能（Skill）加载、会话恢复等交互能力。自定义 `NoAltScreenDriver` 去除终端备用屏幕转义码，使所有输出保留在终端滚动历史中。

### 引擎层（Engine）

- **Agent 循环**：核心编排器，流式接收 LLM 响应 → 收集工具调用 → 按并发安全性分批执行 → 将结果回注 → 循环迭代
- **多模型支持**：统一的 `StreamEvent` 协议，同时支持 Anthropic（Messages API）、OpenAI（Responses API）和 OpenAI 兼容接口（Chat Completions API），切换模型只需改配置
- **Prompt 构建**：优先级排序的 `PromptBuilder`，包含身份、系统环境、任务规范、工具使用、代码风格等多个 Section
- **Token 追踪**：锚点（Anchor）机制——每次 API 返回真实 token 数作为锚点，仅对新增消息做字符估算，实现近零开销的精确 token 跟踪

### 工具层（Tool）

- **25+ 内置工具**：Bash、ReadFile、WriteFile、EditFile、Glob、Grep、Agent、AskUser、LoadSkill、ToolSearch 等
- **并发分批执行**：`partition_tool_calls()` 将连续的安全工具调用归入并行批次，非安全工具顺序执行
- **流式执行**：工具可在 LLM 流式输出过程中提前开始执行，无需等待完整响应
- **MCP 延迟加载**：MCP 工具仅注册名称，LLM 需调用 `ToolSearch` 加载完整 schema，在工具密集场景下减少约 85% 的上下文 token 开销
- **Hook 生命周期**：支持 `startup`、`turn_start`、`pre_tool_use`、`post_tool_use` 等 10 个生命周期事件的自定义 Shell 钩子

### 记忆层（Memory）

- **自动记忆提取**：每隔 N 轮对话，异步调用 LLM 从对话中提取四类记忆：用户偏好、纠正反馈、项目知识、参考资料
- **JSONL 持久化**：记忆按类别存储在 `.mewcode/memory/` 目录，跨会话持久保存
- **Recall 预取**：每次用户消息前，通过 LLM 侧查询相关记忆并注入为系统提示，保持上下文连续性
- **两层上下文压缩**：
  - Layer 1：工具结果预算管控（字符截断 + 磁盘持久化），零 LLM 调用
  - Layer 2：达到上下文窗口 80% 阈值时，触发全对话摘要压缩，保留可配置长度的尾部原文

### 安全层（Security）

五级权限拦截链：危险命令检测 → 路径沙箱 → 规则引擎（用户/项目/本地三级规则） → 权限模式（default/accept-edits/plan/bypass） → 人工确认。任一层级均可拒绝操作，"始终允许"会生成本地规则用于后续会话。

## 多智能体协作

- **子智能体派生**：通过 `Agent` 工具将子任务委派给专用子智能体（如 explore、plan 类型），在后台 `asyncio.Task` 中异步运行
- **团队系统**：`TeamCreate` 创建命名团队，成员通过邮箱机制通信，TUI 实时显示各智能体状态
- **Git Worktree 隔离**：并行智能体使用 `git worktree` 实现文件级隔离，避免编辑冲突
- **Coordinator 模式**：特殊的协调者 prompt，限制主智能体只做任务分解和结果聚合，不直接编码
- **多种启动后端**：支持进程内（inprocess）、iTerm2 新标签页、tmux 面板三种启动方式

## 技术栈

`Python 3.11+` `Textual` `Anthropic SDK` `OpenAI SDK` `MCP` `asyncio` `Pydantic` `Hatchling`
