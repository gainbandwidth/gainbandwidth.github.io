---
title: "MewCode 进阶篇（五）：SubAgent"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "Agent 工具派生子智能体与异步执行"
ShowToc: true
---

## 概述

当一个 Agent 在处理复杂任务时，经常需要"分身"去做一些独立的工作——比如一边让一个子进程去搜索代码库，一边让另一个子进程去跑测试。如果所有工作都在同一个对话上下文中串行执行，不仅效率低下，还会让上下文窗口快速膨胀。

MewCode 的 SubAgent 机制通过 `agents/` 模块实现了一套完整的子智能体系统。它支持通过 Agent 工具派生子智能体、使用 `asyncio.Task` 异步执行、提供 `run_to_completion` 无头模式，并内置 Fork 模式继承对话历史。

## Agent 工具与子智能体派生

当主 Agent 调用 `Agent` 工具时，系统会根据 `subagent_type` 参数查找对应的 Agent 定义（`AgentDef`），创建一个全新的 Agent 实例。这个子 Agent 拥有独立的对话上下文、独立的工具注册表，甚至可以使用不同的模型。

`AgentDef` 通过 YAML frontmatter + Markdown body 的格式定义：

```python
@dataclass
class AgentDef:
    agent_type: str          # 类型名称
    when_to_use: str         # 使用场景描述
    system_prompt: str = ""  # 系统提示
    tools: list[str]         # 白名单工具
    disallowed_tools: list[str]  # 黑名单工具
    model: str = "inherit"   # 模型选择
    max_turns: int = 50      # 最大迭代次数
    background: bool = False # 是否后台执行
    isolation: str = ""      # 隔离模式（worktree）
```

Agent 定义文件可以从三个层级加载，优先级从高到低：

```python
class AgentLoader:
    def load_all(self) -> dict[str, AgentDef]:
        # 优先级 1：项目级（.mewcode/agents/）
        # 优先级 2：用户级（~/.mewcode/agents/）
        # 优先级 3：内置（mewcode/agents/builtins/）
```

`AgentLoader` 还支持**热重载**——每次 `get()` 调用时会检查文件修改时间，如果有变化就重新解析，无需重启应用。

## 工具过滤

子 Agent 不应该拥有和父 Agent 完全相同的工具集。`tool_filter.py` 中的 `resolve_agent_tools()` 实现了四层过滤：

```python
def resolve_agent_tools(parent_registry, definition, is_background):
    # 第 0 层：MCP 工具始终放行
    # 第 1 层：全局禁用（TaskOutput, Agent, AskUserQuestion 等）
    # 第 2 层：自定义 agent 额外限制
    # 第 3 层：后台任务白名单（仅保留 ReadFile, Bash, EditFile 等安全工具）
    # 第 4 层：按 AgentDef 中的 tools/disallowed_tools 过滤
```

全局禁用的工具包括 `Agent`（禁止嵌套派生）、`AskUserQuestion`（子 Agent 不能直接和用户交互）、`Workflow`（防止递归工作流）等。后台任务进一步收紧，只保留文件操作和搜索类工具。

Coordinator 模式有独立的工具集，通过 `apply_coordinator_filter()` 实现：

```python
COORDINATOR_MODE_ALLOWED_TOOLS = frozenset({
    "Agent", "TaskStop", "SendMessage",
    "SyntheticOutput", "TeamCreate", "TeamDelete",
})
```

Coordinator 只能派生和管理子 Agent，不能直接操作文件——这强制它专注于任务分解和结果汇总。

## run_to_completion：无头模式

`run_to_completion` 是 Agent 的"无头"执行模式——不产生流式事件，直接返回最终文本结果。它是子 Agent 执行的核心入口：

```python
async def run_to_completion(
    self, prompt: str,
    conversation: ConversationManager | None = None,
    event_callback: Callable | None = None,
) -> str:
```

与 `run()` 的流式迭代器不同，`run_to_completion` 内部消费所有事件，只收集最终的 assistant 文本。`event_callback` 参数允许外部监听关键事件（如工具调用、token 用量），用于进度追踪。

## TaskManager：后台任务管理

`TaskManager` 负责管理后台运行的子 Agent 任务。每个任务被封装为 `BackgroundTask`：

```python
@dataclass
class BackgroundTask:
    id: str
    name: str
    agent: Agent
    task: str
    status: str = "running"      # running / completed / cancelled / failed
    result: str = ""
    start_time: float
    cancel: Callable[[], None] | None = None
    progress: ProgressInfo
```

`launch()` 方法创建 `asyncio.Task` 在后台执行，并注册取消回调：

```python
def launch(self, agent, task, name="", fork_conversation=None) -> str:
    task_id = uuid.uuid4().hex[:8]
    async_task = asyncio.create_task(
        self._run_background(task_id, fork_conversation)
    )
    bg.cancel = async_task.cancel
    return task_id
```

任务完成后会通过 `asyncio.Queue` 发送通知，主循环通过 `poll_completed()` 消费这些通知，并以 `<task-notification>` 标签注入到对话中。

`adopt_running()` 方法允许接管一个已经在运行的 Agent——当子 Agent 在 worktree 隔离环境中启动时，先由外部启动执行，再被 TaskManager 纳入管理。

## Fork 模式：继承对话历史

Fork 是一种特殊的子 Agent 启动模式——它会**继承父 Agent 的完整对话历史**，让子 Agent 在主 Agent 的上下文基础上继续工作。

`fork.py` 中的 `build_forked_messages()` 负责构建 Fork 后的对话：

```python
def build_forked_messages(conversation, task) -> ConversationManager:
    # 禁止嵌套 Fork
    for msg in conversation.history:
        if FORK_BOILERPLATE_TAG in msg.content:
            raise ForkError("Cannot fork from a forked agent.")

    fork_conv = ConversationManager()
    fork_conv.history = copy.deepcopy(conversation.history)
    # 处理未完成的工具调用
    fork_conv.add_user_message(f"{FORK_BOILERPLATE}\n\n你的任务：\n{task}")
```

Fork 有几个重要的约束：

1. **禁止嵌套**：如果对话历史中已经包含 `FORK_BOILERPLATE_TAG`，说明当前 Agent 本身就是一个 Fork，不允许再 Fork
2. **注入行为约束**：Fork 消息包含一套严格的行为规则——不能再 Fork、不能提问、直接执行工具、结果控制在 500 字以内
3. **未完成的工具调用**：如果父对话最后一条 assistant 消息有未返回结果的工具调用，会用 `"interrupted"` 占位符补齐

## 任务完成通知

子 Agent 完成后，`teammate_tree.py` 中的 `format_task_notification()` 会生成结构化的通知消息：

```python
def format_task_notification(task: BackgroundTask) -> str:
    return (
        f"<task-notification>\n"
        f"Task ID: {task.id}\n"
        f"Agent: {task.name}\n"
        f"Status: {task.status}\n"
        f"Elapsed: {elapsed}\n"
        f"Result:\n{result}\n"
        f"</task-notification>"
    )
```

这些通知以 `user` 角色消息注入到父 Agent 的对话中，父 Agent 可以在下一轮迭代中看到并处理子任务的结果。

## 设计思考

SubAgent 系统的设计反映了几个关键决策：

**独立上下文 vs 共享上下文**：默认的 Agent 工具创建全新对话上下文，避免子 Agent 污染主对话。Fork 模式则反其道行之，适用于需要在已有上下文上继续工作的场景。

**工具集收窄**：子 Agent 的工具集总是父 Agent 的子集。这不是技术限制，而是安全考量——子 Agent 不应该有能力派生更多子 Agent 或直接与用户交互。

**无头模式的价值**：`run_to_completion` 让 Agent 变成了一个可调度的"函数"。Team 系统、后台任务、Worktree 隔离执行都依赖这个模式。它把 Agent 的交互能力和自动化执行能力解耦了。

> 📖 本文是 MewCode 系列的第 12 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
