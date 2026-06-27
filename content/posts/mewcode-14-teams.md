---
title: "MewCode 进阶篇（七）：Agent Teams"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "多 Agent 协作、邮箱通信与协调者模式"
ShowToc: true
---

## 概述

SubAgent 解决了"派生子任务"的问题，但多个子 Agent 之间如何协作？如果一个 Agent 需要知道另一个 Agent 的工作进展，或者它们需要围绕一组共享任务分工合作，单纯的父子关系就不够用了。

MewCode 的 `teams/` 模块实现了一套 Agent 团队协作系统。它通过 `TeamCreate` 创建命名团队，使用基于文件系统的邮箱机制实现跨 Agent 通信，提供 Coordinator 模式让主 Agent 专注于任务分解和结果汇总，并通过 `TeammateTree` TUI 组件实时展示团队状态。

## 团队创建与注册

### TeamManager

`TeamManager` 是团队系统的中枢，管理团队的完整生命周期：

```python
class TeamManager:
    def __init__(self, worktree_manager=None, trace_manager=None):
        self._teams: dict[str, AgentTeam] = {}
        self._task_stores: dict[str, SharedTaskStore] = {}
        self._mailboxes: dict[str, Mailbox] = {}
        self._inprocess_handles: dict[str, InProcessTeammateHandle] = {}
```

### 创建团队

`create_team()` 方法一次性初始化团队所需的全部基础设施：

```python
def create_team(self, name, lead_agent_id, description="") -> AgentTeam:
    backend = self.detect_backend(teammate_mode, is_interactive)
    slug = unique_team_name(name)
    team_dir = resolve_team_dir(slug)

    # 1. 创建团队配置
    team = AgentTeam(name=slug, lead_agent_id=lead_agent_id, ...)
    team.save()

    # 2. 初始化共享任务存储
    task_store = SharedTaskStore(team_dir / "tasks.json")

    # 3. 创建邮箱目录
    mailbox = Mailbox(team_dir / "mailbox")
```

团队配置、共享任务、邮箱——三个子系统各占一个目录，结构清晰。

### 成员注册

`register_member()` 将子 Agent 加入团队，同时注册到全局名称注册表：

```python
def register_member(self, team_name, member: TeammateInfo):
    team.add_member(member)
    team.save()
    AgentNameRegistry.instance().register(member.name, member.agent_id)
```

`AgentNameRegistry` 是一个线程安全的单例，维护名称到 `agent_id` 的映射。`resolve()` 方法支持正向查找（名称 → ID）和反向查找（ID → ID），让 `SendMessage` 工具可以用 Agent 名称而非 UUID 来指定收件人。

## 邮箱通信机制

Agent 之间的通信通过 `Mailbox` 类实现，底层是基于文件系统的消息队列。

### 消息模型

```python
@dataclass
class MailboxMessage:
    id: str
    from_agent: str
    to_agent: str
    content: str
    summary: str = ""
    message_type: str = "text"  # text | shutdown_request | shutdown_response
    timestamp: float = 0.0
    metadata: dict[str, Any]
```

消息类型支持普通文本和关停机协议（`shutdown_request` / `shutdown_response`），后者用于协调团队优雅关闭。

### 文件存储

每个 Agent 有一个独立的邮箱目录，消息以 JSON 文件存储，文件名包含时间戳保证有序：

```python
class Mailbox:
    def write(self, agent_id, message):
        d = self._agent_dir(agent_id)
        filename = f"{message.timestamp:.6f}_{message.id}.json"
        (d / filename).write_text(json.dumps(message.to_dict()))
```

### 消费模式

`consume()` 方法读取并删除所有待处理消息，是典型的"一次消费"语义：

```python
def consume(self, agent_id) -> list[MailboxMessage]:
    messages = []
    for f in sorted(d.iterdir()):
        data = json.loads(f.read_text())
        messages.append(MailboxMessage.from_dict(data))
        f.unlink()  # 读取后删除
    return messages
```

### 广播

`broadcast()` 方法向团队所有成员发送消息，可排除发送者自己：

```python
def broadcast(self, team_members, message, exclude=""):
    for agent_id in team_members:
        if agent_id != exclude:
            self.write(agent_id, message)
```

## 主 Agent 如何消费邮箱

在 `agent.py` 的主循环中，每轮迭代开始前会检查邮箱：

```python
def _consume_mailbox(self, conversation):
    if not self.team_name or not self._team_manager:
        return
    mailbox = self._team_manager.get_mailbox(self.team_name)
    messages = mailbox.consume(self.agent_id)
    for msg in messages:
        prefix = f"[Message from {msg.from_agent}]"
        conversation.add_user_message(f"{prefix} {msg.content}")
```

邮箱消息被注入为 `user` 角色消息，LLM 在下一轮迭代中可以看到并响应这些消息。这是一个非常简洁的设计——不需要额外的消息分发机制，直接复用对话系统的消息管道。

## Coordinator 模式

Coordinator 是一种特殊的 Agent 运行模式——它不直接执行代码修改，而是专注于任务分解、子 Agent 调度和结果汇总。

### 激活条件

```python
def is_coordinator_mode(enable_flag=False) -> bool:
    val = os.environ.get("MEWCODE_COORDINATOR_MODE", "").lower()
    return val in ("1", "true", "yes")
```

通过环境变量激活，也支持 session restore 时自动匹配之前的模式。

### 工具集限制

Coordinator 只能使用一组精心限制的工具：

```python
COORDINATOR_MODE_ALLOWED_TOOLS = frozenset({
    "Agent",          # 派生子 Agent
    "TaskStop",       # 停止子 Agent
    "SendMessage",    # 向子 Agent 发送消息
    "SyntheticOutput",# 输出结构化结果
    "TeamCreate",     # 创建团队
    "TeamDelete",     # 删除团队
})
```

它不能读写文件、不能执行命令——所有实际工作必须委托给子 Agent（Worker）。

### System Prompt 指导

`get_coordinator_system_prompt()` 为 Coordinator 提供了一套详细的行为指南，核心原则包括：

- **合成而非委托理解**：当 Worker 报告研究结果时，Coordinator 必须自己理解后给出具体的实施指令，而不是写"根据你的发现去修复"
- **验证必须独立**：绝不让实现 Worker 验证自己的代码，必须派一个新的 Verification Worker
- **并行是超能力**：读任务可以大量并行，写任务按文件集合串行

### Worker 结果通知

Worker 完成后，结果以 XML 格式的 `<task-notification>` 注入到 Coordinator 的对话中：

```xml
<task-notification>
  <task-id>agent-a1b</task-id>
  <status>completed</status>
  <summary>Agent "Research auth" completed</summary>
  <result>Found null pointer in src/auth/validate.py:42...</result>
</task-notification>
```

Coordinator 通过 `drain_lead_mailbox()` 收集所有团队通知，注入为 `<team-notification>` 标签。

## 共享任务系统

`SharedTaskStore` 提供了一套基于 JSON 文件的共享任务管理：

```python
@dataclass
class SharedTask:
    id: str
    title: str
    description: str = ""
    status: str = "pending"    # pending | in_progress | completed | blocked
    assignee: str = ""
    blocks: list[str]          # 阻塞哪些任务
    blocked_by: list[str]      # 被哪些任务阻塞
    created_by: str = ""
```

每次读写都从文件加载最新状态（`_load()`），保证多个 Agent 通过文件系统实现最终一致性。通过 `TaskCreate`、`TaskGet`、`TaskList`、`TaskUpdate` 四个工具，Agent 可以协作管理任务列表。

## TeammateTree：TUI 实时状态

`teammate_tree.py` 实现了一个基于 Textual 的 TUI 组件，实时展示团队成员的执行状态：

```python
class TeammateTree(Widget):
    teammates: reactive[list[TeammateProgress]] = reactive(list)

    def render(self) -> Text:
        lines = Text()
        lines.append("  ┌─ team-lead: thinking…\n")
        for i, p in enumerate(self.teammates):
            connector = "  └─ " if is_last else "  ├─ "
            lines.append(f"{connector}@{p.name}: {p.activity_summary}")
            lines.append(f" · {p.tool_use_count} tools")
```

树形展示以 `team-lead` 为根节点，每个子 Agent 显示当前活动（如 "Reading src/auth.py"）、工具调用次数和 token 用量。`TeammateProgress` 对象通过 `event_callback` 实时更新——每次工具调用和 token 统计都会触发 TUI 刷新。

有趣的细节是 `SPINNER_VERBS`——每个子 Agent 启动时会随机分配一个动词（如 "Marinating"、"Cogitating"、"Moonwalking"），在等待状态时显示，增加了一点趣味性。

## 启动后端

团队支持三种执行后端，通过 `backend_detect.py` 自动探测：

### InProcess（默认）

子 Agent 在同一个 Python 进程中通过 `asyncio.Task` 运行：

```python
def spawn_inprocess_teammate(agent, prompt, name, ...) -> InProcessTeammateHandle:
    progress = TeammateProgress(name=name, team_name=team_name)
    task = asyncio.create_task(_run(), name=f"teammate-{name}")
    return InProcessTeammateHandle(agent, task, name, progress)
```

这是默认后端，优势是可以实时追踪进度（`TeammateProgress`），劣势是所有 Agent 共享同一个进程。

### tmux

在 tmux 的新 pane 中启动独立的 mewcode CLI 进程：

```python
def spawn_tmux_teammate(team_name, teammate_name, worktree_path, prompt, ...):
    pane_id = _run_tmux("split-window", "-h", "-P", "-F", "#{pane_id}")
    cli_cmd = build_cli_command(team_name, teammate_name, worktree_path, prompt)
    _run_tmux("send-keys", "-t", pane_id, cli_cmd, "Enter")
```

通过环境变量 `MEWCODE_TEAM_NAME` 和 `MEWCODE_TEAMMATE_NAME` 将 CLI 进程关联到团队。

### iTerm2

使用 `it2` CLI 工具在 iTerm2 的分屏中启动：

```python
def spawn_iterm2_teammate(team_name, teammate_name, worktree_path, prompt, ...):
    session_id = _run_it2("split-pane", "--command", f"/bin/zsh -c '{cli_cmd}'")
```

后端选择逻辑：如果用户没有显式指定，默认使用 InProcess。当需要 tmux/iTerm2 分屏展示时，按 tmux → iTerm2 的优先级探测可用终端。

## 团队清理

`delete_team()` 执行完整的团队清理流程：

```python
def delete_team(self, team_name):
    # 1. 检查是否有活跃成员
    # 2. 注销名称注册表
    # 3. 取消 InProcess 任务 / 杀死 tmux pane
    # 4. 清理 worktree
    # 5. 清理邮箱目录
    # 6. 删除团队目录
```

清理前会检查所有成员是否已停止——如果有活跃成员，拒绝删除。

## 设计思考

**邮箱 vs 直接调用**：文件系统邮箱看似"原始"，但它天然支持跨进程通信（tmux/iTerm2 后端），且消息持久化、可审计。相比内存队列或 gRPC，它更简单也更健壮。

**Coordinator 的能力收窄**：限制 Coordinator 只做调度不做实现，不是技术限制，而是角色分离。这避免了"既当裁判又当运动员"的问题——Coordinator 不会因为自己写了代码而产生确认偏误。

**InProcess 优先**：默认使用进程内执行而非 tmux，因为实时进度追踪对用户体验至关重要。tmux/iTerm2 后端保留了在独立终端窗口中观察 Agent 的能力，适合调试和演示场景。

> 📖 本文是 MewCode 系列的第 14 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
