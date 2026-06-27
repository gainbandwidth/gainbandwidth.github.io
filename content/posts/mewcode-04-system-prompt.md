---
title: "MewCode 骨架篇（四）：System Prompt 设计"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "拆解 MewCode 系统提示词的优先级排序与分段组织设计"
ShowToc: true
---

## 概述

System Prompt 是 Agent 行为的"宪法"——它决定了 AI 如何理解自身角色、如何处理任务、何时该谨慎、何时该果断。MewCode 的 `prompts.py` 用不到 300 行代码，构建了一套结构清晰、可扩展的系统提示词体系。本文将从三个维度拆解这套设计：**PromptBuilder 的优先级排序机制**、**各 Section 的组织逻辑**、以及 **Plan Mode 的提醒策略**。

## 核心数据结构：PromptSection 与 PromptBuilder

一切始于一个极简的 dataclass：

```python
@dataclass
class PromptSection:
    name: str
    priority: int
    content: str
```

每个 Section 是提示词的一个"块"，带有名称、优先级数值和文本内容。`PromptBuilder` 负责将这些块组装成最终 prompt：

```python
class PromptBuilder:
    def add(self, section: PromptSection) -> PromptBuilder:
        self._sections.append(section)
        return self

    def build(self) -> str:
        self._sections.sort(key=lambda s: s.priority)
        parts = [s.content.strip() for s in self._sections if s.content.strip()]
        return "\n\n".join(parts)
```

关键设计点：

1. **优先级排序而非插入顺序排序** —— `build()` 方法在拼接前按 `priority` 升序排列，这意味着各 Section 的添加顺序无关紧要，最终输出始终由 priority 数值决定。
2. **链式调用** —— `add()` 返回 `self`，支持 `b.add(A).add(B).add(C)` 的流畅写法。
3. **空内容过滤** —— `if s.content.strip()` 自动跳过空白 Section，避免输出中出现多余空行。

## 八大 Section：从身份到环境的渐进式组织

MewCode 将系统提示词拆分为 8 个核心 Section，优先级从 0 到 70 递增，后续还有 3 个可选 Section（优先级 80-95）。整体排列遵循一个原则：**越基础、越不可变的规则越靠前，越动态、越个性化的内容越靠后**。

| Section | Priority | 职责 |
|---|---|---|
| Identity | 0 | 定义 Agent 身份与安全底线 |
| System | 10 | 工具调用规则与上下文机制 |
| DoingTasks | 20 | 任务执行的行为准则 |
| ExecutingActions | 30 | 操作风险评估与确认策略 |
| UsingTools | 40 | 工具使用的具体规范 |
| ToneStyle | 50 | 输出风格与语气约束 |
| TextOutput | 60 | 文本输出的沟通原则 |
| Environment | 70 | 运行时环境信息（动态生成） |
| CustomInstructions | 80 | 用户自定义项目指令（可选） |
| Skills | 90 | 活跃技能注入（可选） |
| Memory | 95 | 记忆上下文注入（可选） |

### Identity（priority=0）：身份锚定与安全红线

Identity 放在最高优先级（数值最低），因为它定义了 Agent 是谁、什么绝对不能做。这段内容包含两条 `IMPORTANT` 指令：不引入安全漏洞（命令注入、XSS、SQL 注入等），以及不凭空生成 URL。这些是无论后续任何 Section 都不能覆盖的底线规则。

### System（priority=10）：运行时行为框架

System Section 规定了工具调用的权限模型（用户拒绝后不得重试同一调用）、`<system-reminder>` 标签的含义、prompt injection 的防范意识，以及自动摘要带来的"无限上下文"能力。这些信息为 Agent 提供了运行时的"物理法则"。

### DoingTasks（priority=20）：任务执行的"工匠精神"

这是篇幅最长的 Section，涵盖了 MewCode 对代码工程的核心哲学：先读后改、偏好编辑而非创建、失败时先诊断再换策略、不做过度抽象、不写无意义注释、完成任务必须验证。每一条规则都对应着真实场景中 Agent 容易犯的错误。

值得注意的是"exploratory questions"的处理——对于探索性问题，Agent 应给出 2-3 句简短建议并等待用户确认，而非直接动手实现。这是一种**主动刹车**机制，避免 Agent 在模糊需求下做出不可逆操作。

### ExecutingActions（priority=30）：风险分级

这个 Section 将操作按"可逆性"和"影响范围"分为两类：本地可逆操作（编辑文件、跑测试）可以自由执行；涉及破坏性、难逆转或对外可见的操作（force-push、删除分支、创建 PR）则需要用户确认。核心思想是：**Agent 应当像谨慎的工程师一样，对高风险操作保持敬畏**。

### UsingTools（priority=40）与 ToneStyle（priority=50）

UsingTools 强制 Agent 使用专用工具而非 Bash 替代（用 `ReadFile` 而非 `cat`，用 `Grep` 而非 `grep`），目的是让操作对用户透明可审查。ToneStyle 则控制输出风格：简短、不用 emoji、引用代码时附带 `file_path:line_number` 格式的定位信息。

### Environment（priority=70）：动态注入

与其他静态 Section 不同，Environment 通过 `environment_section()` 函数动态生成，注入当前工作目录、操作系统版本和日期：

```python
def environment_section(work_dir: str) -> PromptSection:
    lines = [
        "# Environment",
        f" - Working directory: {work_dir}",
        f" - Platform: {platform.system()} {platform.release()}",
        f" - Date: {datetime.now().strftime('%Y-%m-%d')}",
    ]
    return PromptSection(name="Environment", priority=70, content="\n".join(lines))
```

这是唯一必须依赖运行时状态的 Section。

## 可选 Section 与 Hook 注入

`build_system_prompt()` 函数支持三个可选 Section：

```python
if custom_instructions:
    b.add(PromptSection(name="CustomInstructions", priority=80, ...))
if skill_section:
    b.add(PromptSection(name="Skills", priority=90, ...))
if memory_section:
    b.add(PromptSection(name="Memory", priority=95, ...))
```

这三个 Section 的优先级最低（数值最高），确保它们不会覆盖核心规则。此外，Hook 注入的提示词直接拼接在最终 prompt 末尾，完全绕过了优先级系统——这是一种有意的设计：Hook 内容来自外部插件，不应参与内部排序逻辑。

## Plan Mode 提醒策略

Plan Mode 是 MewCode 的"只读规划"模式，Agent 在此模式下只能读取代码和编写计划文件，不能执行任何修改操作。`prompts.py` 为此设计了两种提醒模板：

- **Full Reminder**（`_PLAN_MODE_FULL_REMINDER`）：包含完整的 5 阶段工作流说明——Initial Understanding → Design → Review → Final Plan → ExitPlanMode，指导 Agent 使用 `explore` 子代理探索代码库、使用 `plan` 子代理设计方案。
- **Sparse Reminder**（`_PLAN_MODE_SPARSE_REMINDER`）：仅一行，提醒 Agent 仍处于 Plan Mode 并指向计划文件路径。

`build_plan_mode_reminder()` 通过 `_REMINDER_INTERVAL = 5` 控制提醒密度：

```python
def build_plan_mode_reminder(plan_path, plan_exists, iteration):
    if iteration == 1:
        return _PLAN_MODE_FULL_REMINDER.format(...)
    attachment_index = (iteration - 1) // _REMINDER_INTERVAL
    if attachment_index % _REMINDER_INTERVAL == 0:
        return _PLAN_MODE_FULL_REMINDER.format(...)
    return _PLAN_MODE_SPARSE_REMINDER.format(...)
```

首轮对话给出完整指引，之后每 5 轮重复一次完整提醒，中间轮次仅给出简短提示。这种**稀疏提醒**策略在"保持 Agent 不偏离模式"和"避免浪费 token 重复长文本"之间取得了平衡。

## 设计思考与取舍

**为什么用数值优先级而非枚举/列表顺序？** 数值优先级允许新增 Section 时无需修改已有代码——只需选一个合适的数值即可插入任意位置。如果使用列表顺序，每次插入都需要修改中心化的排列逻辑，违反了开闭原则。

**为什么 Hook 内容不走优先级系统？** Hook 是外部扩展点，其内容不可预测。将其置于末尾是一种安全隔离：即使 Hook 内容包含与核心规则冲突的指令，LLM 也会更倾向于遵循出现在 prompt 前部的核心规则（recency bias 的反向利用）。

**Coordinator Mode 的短路设计。** `build_system_prompt()` 在 `coordinator_mode=True` 时直接返回完全不同的提示词，跳过了所有 Section 的构建。这说明 Coordinator（多 Agent 协调者）的角色定位与普通 Agent 差异极大，复用同一套 Section 反而会增加复杂度。

> 📖 本文是 MewCode 系列的第 04 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
