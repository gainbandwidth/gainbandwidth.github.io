---
title: "MewCode 骨架篇（七）：上下文管理"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "两层压缩策略与断路器机制，让 Agent 在有限窗口中持续工作"
ShowToc: true
---

## 概述

LLM 的上下文窗口是有限的——200K tokens 听起来很大，但一次包含大量工具调用的 coding session 可能很快就会把它填满。一旦溢出，要么请求失败，要么模型被迫截断输入，丢失关键上下文。

MewCode 的 `mewcode.context` 模块用一套**两层压缩**策略解决这个问题：Layer 1 在不触发任何 LLM 调用的前提下管控工具结果的体积；Layer 2 在逼近上下文上限时启动全对话摘要压缩。两者配合 `ContentReplacementState`、`RecoveryState` 和断路器机制，让 Agent 能在有限窗口里持续、稳定地工作。

## Layer 1：工具结果预算管控（零 LLM 调用）

Layer 1 的核心入口是 `apply_tool_result_budget()`。它遍历整个对话历史，对每条 `ToolResultBlock` 做三轮处理，全程只做字符串操作和文件 I/O，不调用 LLM。

### Pass 1：单条超限检测

任何一条工具结果超过 `SINGLE_RESULT_CHAR_LIMIT`（50K 字符），就会被落盘保存为文件，原位替换成一段 2KB 预览加文件路径的 `<persisted-output>` 标签：

```python
if len(tr.content) > SINGLE_RESULT_CHAR_LIMIT:
    fp = persist_tool_result(tr.tool_use_id, tr.content, session_dir)
    preview = make_persisted_preview(tr.content, fp)
    decisions[tr.tool_use_id] = preview
```

模型看到预览后，如果需要完整内容可以用文件读取工具重新加载。

### Pass 2：聚合超限检测

即使每条结果都没超过单条上限，它们的总量也可能超标。当所有 `decisions` 的字符总和超过 `AGGREGATE_CHAR_LIMIT`（200K 字符）时，按体积从大到小排序，逐个落盘直到总量回到预算内：

```python
ranked = sorted(remaining, key=lambda tr: len(tr.content), reverse=True)
for tr in ranked:
    if total <= AGGREGATE_CHAR_LIMIT:
        break
    fp = persist_tool_result(tr.tool_use_id, tr.content, session_dir)
    # ...
    total -= old_len - len(preview)
```

### Pass 3：陈旧结果裁剪

`_snip_stale_messages()` 负责处理历史深处的旧工具结果。对话超过 `KEEP_RECENT_TURNS`（10 轮）之前的结果，如果还保留着原始内容，会被裁剪为 200 字符预览加 `<snipped>` 标记。近期消息则保持原样——这体现了一个核心思路：**越新的上下文越重要，越旧的上下文越可以被压缩**。

### ContentReplacementState：决策冻结

Layer 1 引入了 `ContentReplacementState` 来保证替换决策的稳定性。它包含两个字段：

- `seen_ids`：已处理过的 `tool_use_id` 集合
- `replacements`：已决定替换内容的映射表

关键设计是 **Design B（决策冻结）**：一旦某个 `tool_use_id` 被决定替换（或保留原文），这个决策就不会再改变。`apply_tool_result_budget()` 不 mutate 原始 `ConversationManager`，而是返回一个新实例，但 `state` 本身会被 mutate——新决定的 id 进入 `seen_ids`，新替换进入 `replacements`。

这些决策通过 `append_replacement_records()` 以 JSONL 格式持久化到 session 目录。当会话 resume 时，`reconstruct_replacement_state()` 从磁盘记录重建状态，确保同一条工具结果不会因为 resume 而"复活"为完整原文。

## Layer 2：全对话摘要压缩

当 Layer 1 已经尽力压缩了工具结果，但上下文仍然逼近窗口上限时，Layer 2 介入。

### 触发阈值

`compute_compact_threshold()` 的计算公式为：

```
threshold = context_window - SUMMARY_OUTPUT_RESERVE(20K) - AUTO_COMPACT_SAFETY_MARGIN(13K)
```

对于 200K 窗口，阈值约为 167K tokens。`auto_compact()` 通过 `conversation.current_tokens()` 获取当前用量（以上次 API 计费为锚点加增量估算），超过阈值就启动压缩。

### 前缀摘要 + 尾部保留

压缩不是对整个对话做摘要，而是只摘要**前缀**，保留**尾部原文**。`_compute_keep_start_index()` 从尾部向头部遍历，按 token 累加：

- 保底条件：至少保留 `KEEP_RECENT_TOKENS`（10K tokens）或 `MIN_KEEP_MESSAGES`（5 条），取先满足的
- 硬上限：不超过 `KEEP_MAX_TOKENS`（40K tokens），防止单条大消息吞掉整个窗口
- 对齐修正：`_align_keep_start_to_tool_pair()` 确保 `tool_use` 和对应的 `tool_result` 不会被拆散

前缀太小时（低于 `MIN_SUMMARIZE_PREFIX_TOKENS` 即 2K tokens），`_prefix_too_small_to_compact()` 会返回 True，退化为不压缩——因为摘要本身的 LLM 调用开销可能比回收的空间还大。

### 摘要生成

`SUMMARY_PROMPT` 要求模型输出结构化摘要，包含 9 个部分：用户意图、技术概念、文件与代码、错误与修复、解决过程、所有用户消息（原文保留）、待办任务、当前工作、下一步计划。先输出 `<analysis>` 标签做思考（会被丢弃），再输出 `<summary>` 标签做正式摘要。`extract_summary()` 只提取 `<summary>` 标签内的内容。

摘要生成有 3 次重试机制。如果因 prompt 过长失败，会按 turn 分组后丢弃前 20% 的内容再重试。

### 压缩后重建

`build_compact_messages()` 将摘要包装为一条 user 消息，附上恢复附件（下一节详述），并告知模型完整会话记录的 transcript 路径——如果模型需要压缩前的细节，可以自行用文件读取工具加载。

## RecoveryState：压缩后的上下文恢复

压缩会清空工作对话，模型会丢失很多"正在使用中"的上下文：刚读过的文件内容、正在执行的 skill SOP、可用的工具列表。`RecoveryState` 解决的就是这个问题。

它是一个线程安全的 per-agent 快照，记录两类信息：

- **文件读取记录**（`FileReadRecord`）：每次 `ReadFile` 工具返回时调用 `record_file_read()`
- **技能调用记录**（`SkillInvocationRecord`）：每次 skill 被激活时调用 `record_skill_invocation()`

压缩后，`build_recovery_attachment()` 将这些快照渲染为摘要消息的附件，包含四个小节：

1. **最近读过的文件**：最多 `RECOVERY_FILE_LIMIT`（5 个）文件，每个截断到 `RECOVERY_TOKENS_PER_FILE`（5K tokens）
2. **已激活的技能**：按时间倒序，总预算 `RECOVERY_SKILLS_BUDGET`（25K tokens），每个技能截断到 5K tokens
3. **可用工具**：列出当前接入的工具名称和描述
4. **提示**：提醒模型这些是重建的上下文，如需原文应重新读取

这样，即便对话被压缩到只剩一条摘要消息，模型仍然知道自己在做什么、有哪些资源可用。

## 断路器机制

`CompactCircuitBreaker` 是一个简洁的断路器实现：

```python
@dataclass
class CompactCircuitBreaker:
    max_failures: int = 3
    consecutive_failures: int = field(default=0, init=False)

    def record_failure(self) -> None:
        self.consecutive_failures += 1

    def record_success(self) -> None:
        self.consecutive_failures = 0

    def is_open(self) -> bool:
        return self.consecutive_failures >= self.max_failures
```

当 `auto_compact()` 连续失败 3 次（摘要请求因各种原因失败），断路器"熔断"，后续自动压缩请求直接跳过并返回提示信息。用户可以手动执行 `/compact` 命令来尝试手动压缩。一次成功的压缩会调用 `record_success()` 重置计数器。

这个机制防止了在网络异常或 API 限流时，每次对话轮次都重复触发注定失败的压缩请求。

## 设计思考与取舍

**为什么 Layer 1 不调用 LLM？** 工具结果压缩是一个高频操作，每次对话轮次都可能触发。如果每次都调用 LLM 来决定怎么压缩，压缩本身的延迟和 token 成本就不可接受。用规则（字符数阈值 + 排序贪心）做决策，能在微秒级完成。

**为什么用"替换预览"而不是"直接删除"？** 直接删除工具结果会让模型看到不完整的对话——它发起了一个工具调用，但看不到任何结果。保留预览加文件路径，让模型知道结果存在且可以重新获取，这比信息黑洞要好得多。

**ContentReplacementState 为什么不跟着 ConversationManager 一起重建？** 这是一个有意的设计。替换决策需要跨 resume 持久化，否则 resume 后同样的大结果会被"恢复"为原文，再次触发压缩，形成无意义的循环。通过 JSONL 文件独立持久化，`reconstruct_replacement_state()` 在 resume 时重建映射关系，保证决策的连续性。

**为什么摘要只压缩前缀而不是全部？** 近期消息包含模型当前的工作上下文——正在修改的文件、还没完成的步骤、用户最新的指令。对这些消息做摘要是高损的，模型很可能丢失关键细节。只摘要远处的历史、保留近处的原文，在压缩率和信息保真度之间取得平衡。

> 本文是 MewCode 系列的第 07 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
