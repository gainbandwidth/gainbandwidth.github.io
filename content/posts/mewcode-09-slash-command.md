---
title: "MewCode 进阶篇（二）：Slash Command"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "斜杠命令系统的注册、分发与执行机制"
ShowToc: true
---

## 概述

在终端 TUI 中，用户除了与 AI 对话，还需要执行各种本地操作：清除历史、压缩上下文、切换权限模式、管理会话。MewCode 设计了一套**斜杠命令系统**，将 `/help`、`/compact`、`/clear`、`/session` 等操作统一抽象为可注册、可发现的 `Command` 对象，通过 `CommandRegistry` 集中管理，支持别名、自动补全和动态注册。

## 命令解析

一切从用户输入开始。`parse_command()` 负责判断输入是否为命令，并拆分出命令名和参数：

```python
def parse_command(text: str) -> tuple[str, str, bool]:
    text = text.strip()
    if not text.startswith("/"):
        return "", "", False
    parts = text[1:].split(None, 1)
    name = parts[0].lower()
    args = parts[1].strip() if len(parts) > 1 else ""
    return name, args, True
```

返回值是一个三元组 `(name, args, is_command)`。`/compact 保留最近 5 轮` 会被解析为 `("compact", "保留最近 5 轮", True)`。

## 命令注册与 CommandRegistry

每个命令是一个 `Command` 数据类，包含名称、描述、类型、处理函数和可选的别名列表：

```python
@dataclass
class Command:
    name: str
    description: str
    type: CommandType         # LOCAL / LOCAL_UI / PROMPT
    handler: CommandHandler   # async (CommandContext) -> None
    aliases: list[str] = field(default_factory=list)
    usage: str = ""
    arg_prompt: str = ""
    hidden: bool = False
```

`CommandType` 区分了三种执行方式：

| 类型 | 含义 | 示例 |
|------|------|------|
| `LOCAL` | 纯本地逻辑，不触发 UI 更新 | `/help`、`/session list` |
| `LOCAL_UI` | 本地逻辑 + UI 刷新 | `/clear`、`/plan` |
| `PROMPT` | 将内容作为 Prompt 发送给 Agent | 自定义命令 |

`CommandRegistry` 是命令的注册中心，维护命令表和别名映射，支持同步和异步两种注册方式：

```python
class CommandRegistry:
    def __init__(self):
        self._commands: dict[str, Command] = {}
        self._alias_map: dict[str, str] = {}

    def register_sync(self, command: Command) -> None:
        self._commands[command.name] = command
        for alias in command.aliases:
            self._alias_map[alias] = command.name

    def find(self, name: str) -> Command | None:
        if name in self._commands:
            return self._commands[name]
        canon = self._alias_map.get(name)
        if canon:
            return self._commands.get(canon)
        return None
```

## CommandContext：依赖注入

每个命令的 handler 接收一个 `CommandContext`，其中包含了运行所需的所有依赖：

```python
@dataclass
class CommandContext:
    args: str             # 命令参数
    agent: Any            # Agent 实例
    conversation: Any     # ConversationManager
    session: Any          # 当前 Session
    session_manager: Any  # SessionManager
    memory_manager: Any   # MemoryManager
    ui: UIController      # UI 控制接口
    config: Any           # 扩展配置字典
```

这是一种轻量级的依赖注入——命令不需要知道各个组件如何初始化，只需要从 context 中取用。`UIController` 是一个 Protocol，定义了 `add_system_message`、`send_user_message`、`set_plan_mode` 等方法，解耦了命令逻辑和具体的 TUI 实现。

## 核心命令分析

### /compact — 上下文压缩

```python
async def handle_compact(ctx: CommandContext) -> None:
    input_tokens, _ = ctx.ui.get_token_count()
    if input_tokens < 5000:
        ctx.ui.add_system_message(f"当前 token 数 {input_tokens:,}，无需压缩")
        return
    result = await ctx.agent.manual_compact(ctx.conversation)
    if isinstance(result, CompactNotification):
        if ctx.session is not None and result.boundary is not None:
            ctx.session.append_record(
                make_compact_boundary(result.boundary.summary, result.boundary.keep)
            )
```

压缩前先检查 token 数——低于 5000 就没有压缩的必要。压缩成功后会写入一条 `compact_boundary` 记录，使得会话恢复时能直接从压缩点重建。

### /clear — 清除对话

```python
async def handle_clear(ctx: CommandContext) -> None:
    if ctx.session:
        ctx.session.close()
    new_session = ctx.session_manager.create()
    ctx.config["set_session"](new_session)
    ctx.config["set_conversation"](ConversationManager())
    if ctx.agent:
        ctx.agent._loop_count = 0
        ctx.agent.clear_active_skills()
```

清除不只是清空 UI——它关闭旧 Session、创建新 Session、重置 ConversationManager、清零 Agent 的循环计数，并清除已激活的 Skill。这是一个完整的"重新开始"。

### /session — 会话管理

`/session` 是一个多子命令的设计，支持 `list`、`resume`、`new`、`delete` 四种操作：

```python
async def handle_session(ctx: CommandContext) -> None:
    sub = parts[0] if parts else ""
    if sub == "list":
        metas = sm.list()
        # 展示最近 10 个会话
    elif sub == "resume":
        result = sm.resume(session_id)
        # 恢复对话历史、重建 ConversationManager
    elif sub == "new":
        # 关闭旧会话、创建新会话
    elif sub == "delete":
        # 不能删除当前活跃会话
```

`resume` 特别值得注意——它支持通过序号选择候选会话，也支持直接输入 session ID，恢复时会重新渲染历史消息到 UI。

## 自动补全

命令系统内置了 Tab 补全支持。`complete()` 函数在命令注册表中前缀匹配，返回最多 8 个候选项：

```python
def complete(registry: CommandRegistry, prefix: str) -> list[tuple[str, str]]:
    for cmd in registry.list_commands():
        if cmd.name.startswith(prefix) or any(a.startswith(prefix) for a in cmd.aliases):
            display = f"/{cmd.name:<16} — {desc}"
            matches.append((display, "/" + cmd.name))
```

TUI 层的 `ChatInput` 监听 `on_text_area_changed` 事件，当用户输入以 `/` 开头且不含空格时，自动触发补全弹窗。

## 动态注册：Skill 命令

除了启动时的静态注册（`register_all_commands`），命令系统还支持运行时动态注册。Skill 系统加载后，会通过 `register_skill_commands()` 为每个 Skill 注册对应的斜杠命令，使用户可以直接 `/review`、`/commit` 等调用技能包。

## 设计思考

斜杠命令系统的设计哲学是**统一入口、分散实现**。用户不需要记住"按哪个快捷键做什么"，只需要知道"以 `/` 开头的就是命令"。`CommandType` 的三种类型将 UI 交互与业务逻辑解耦，`CommandContext` 的依赖注入让每个命令可以独立测试。这种设计也为后续的插件化打下了基础——只要实现 `CommandHandler` 接口，任何模块都可以向系统注册新命令。

> 📖 本文是 MewCode 系列的第 09 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
