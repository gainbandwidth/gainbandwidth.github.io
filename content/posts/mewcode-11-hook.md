---
title: "MewCode 进阶篇（四）：Hook 系统"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "生命周期事件驱动，Shell 钩子与工具拦截"
ShowToc: true
---

## 概述

在 AI Agent 的主循环中，很多时候我们希望在特定节点"插入"自定义逻辑——比如每次工具调用前做一次权限检查，或者在会话结束时自动发送通知。如果把这些逻辑硬编码到 Agent 循环里，每增加一个场景就要改一次代码。

MewCode 的 `mewcode.hooks` 模块用一套**事件-动作**模型解决了这个问题。它定义了 10 个核心生命周期事件，允许用户通过配置文件声明"在某个事件发生时执行某个动作"，而不需要修改 Agent 的任何代码。更关键的是，Hook 可以**拦截工具调用**——在 `pre_tool_use` 事件中返回拒绝，直接阻止工具执行。

## 生命周期事件

`LifecycleEvent` 枚举定义了所有可监听的事件，按粒度分为四个层级：

```python
class LifecycleEvent(StrEnum):
    # 系统级
    STARTUP = "startup"
    SHUTDOWN = "shutdown"

    # 会话级
    SESSION_START = "session_start"
    SESSION_END = "session_end"

    # 轮次级
    TURN_START = "turn_start"
    TURN_END = "turn_end"

    # 消息级
    PRE_SEND = "pre_send"
    POST_RECEIVE = "post_receive"

    # 工具级
    PRE_TOOL_USE = "pre_tool_use"
    POST_TOOL_USE = "post_tool_use"
```

这 10 个事件覆盖了 Agent 从启动到关闭的完整生命周期。事件触发顺序是：`startup` → `session_start` → (`turn_start` → `pre_send` → `post_receive` → `pre_tool_use`/`post_tool_use` → `turn_end`) 循环 → `session_end` → `shutdown`。

此外还有一些辅助事件（`error`、`compact`、`file_change` 等），用于更细粒度的场景监听。

## 核心模型

Hook 系统的核心模型在 `models.py` 中定义，由三个数据类构成：

### Action：要做什么

```python
@dataclass
class Action:
    type: str          # command / prompt / http / agent
    command: str = ""  # Shell 命令
    message: str = ""  # 注入消息
    url: str = ""      # HTTP 端点
    timeout: int = 30
```

支持四种动作类型：执行 Shell 命令（`command`）、向 Agent 注入提示消息（`prompt`）、发送 HTTP 请求（`http`）和调用另一个 Agent（`agent`，预留）。

### Hook：事件绑定

```python
@dataclass
class Hook:
    id: str
    event: str
    action: Action
    condition: ConditionGroup | None = None
    reject: bool = False       # 是否拒绝工具调用
    once: bool = False         # 是否只执行一次
    async_exec: bool = False   # 是否异步执行
    executed: bool = False
```

关键字段是 `reject`——只有 `pre_tool_use` 事件可以使用它。当 `reject=True` 且 Shell 命令返回非零退出码时，工具调用会被拦截，抛出 `ToolRejectedError`。

### HookContext：运行时上下文

`HookContext` 携带当前事件的运行时信息——工具名称、参数、文件路径、消息内容等。它提供 `expand()` 方法，将 Action 中的模板变量（`$TOOL_NAME`、`$FILE_PATH` 等）替换为实际值：

```python
def expand(self, template: str) -> str:
    result = template
    result = result.replace("$EVENT", self.event_name)
    result = result.replace("$TOOL_NAME", self.tool_name)
    result = result.replace("$FILE_PATH", self.file_path)
    # ...
```

## HookEngine：调度核心

`HookEngine` 是 Hook 系统的调度中心，负责匹配和执行 Hook。

### 匹配逻辑

`find_matching_hooks()` 按三个条件过滤：事件名称匹配、`should_run()` 检查（处理 `once` 语义）、`condition` 条件评估：

```python
def find_matching_hooks(self, event: str, ctx: HookContext) -> list[Hook]:
    for hook in self.hooks:
        if hook.event != event:
            continue
        if not hook.should_run():
            continue
        if hook.condition is not None and not hook.condition.evaluate(ctx):
            continue
        matched.append(hook)
    return matched
```

### 工具拦截

`run_pre_tool_hooks()` 是 Hook 系统最核心的能力之一。它在工具执行前运行，如果某个 Hook 的 `reject=True` 且动作执行成功，就返回 `ToolRejectedError`：

```python
async def run_pre_tool_hooks(self, ctx: HookContext) -> ToolRejectedError | None:
    for hook in matched:
        result = await execute_action(hook.action, ctx)
        if hook.reject:
            return ToolRejectedError(
                tool=ctx.tool_name,
                reason=result.output,
                hook_id=hook.id,
            )
    return None
```

在 Agent 主循环中，当 `run_pre_tool_hooks` 返回非 `None` 时，工具执行会被跳过，错误信息作为工具结果返回给 LLM：

```python
rejection = await self.hook_engine.run_pre_tool_hooks(hook_ctx)
if rejection is not None:
    result = ToolResult(
        output=f"Hook rejected: {rejection.reason}",
        is_error=True,
    )
```

### Prompt 注入

`prompt` 类型的 Action 会将输出收集到 `_prompt_messages` 列表中。在下一轮 LLM 调用前，`build_system_prompt()` 会将这些消息注入到 System Prompt 中：

```python
if hook.action.type == "prompt" and result.success:
    self._prompt_messages.append(result.output)
```

这允许 Hook 在不修改对话历史的情况下，向 LLM 注入临时指令或上下文提示。

## 条件表达式

`conditions.py` 实现了一套轻量级的条件表达式解析器，支持四种比较运算符：

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `==` | 精确匹配 | `tool == "Bash"` |
| `!=` | 不等于 | `tool != "ReadFile"` |
| `=~` | 正则匹配 | `args.command =~ /rm\s+-rf/` |
| `~=` | Glob 匹配 | `file_path ~= "*.py"` |

多个条件可以用 `&&`（AND）或 `||`（OR）组合，但不能混用：

```python
def parse_condition(expr: str) -> ConditionGroup | None:
    if has_and and has_or:
        raise ConditionParseError(
            "Cannot mix '&&' and '||' in a single condition expression."
        )
```

这个限制是有意的——避免引入复杂的优先级规则，保持配置的可读性。如果需要更复杂的逻辑，可以拆分成多个 Hook。

## 执行器

`executors.py` 实现了四种动作的执行逻辑：

- **`execute_command`**：通过 `asyncio.create_subprocess_shell` 执行 Shell 命令，支持超时控制
- **`execute_prompt`**：直接将消息文本加入注入队列
- **`execute_http`**：使用 `urllib` 发送 HTTP 请求（在线程池中执行，避免阻塞事件循环）
- **`execute_agent`**：预留的 Agent 调用接口

Shell 命令执行器有完善的超时保护：

```python
async def execute_command(action: Action, ctx: HookContext) -> ActionResult:
    command = ctx.expand(action.command)
    proc = await asyncio.create_subprocess_shell(command, ...)
    try:
        stdout, _ = await asyncio.wait_for(proc.communicate(), timeout=action.timeout)
    except asyncio.TimeoutError:
        proc.kill()
        return ActionResult(output=f"Command timed out", success=False)
```

## 配置加载

`loader.py` 中的 `load_hooks()` 负责将原始字典列表解析为 `Hook` 对象，并做严格的校验：

- 事件名必须在 `LifecycleEvent` 中
- `reject` 只能与 `pre_tool_use` 搭配
- `async` 不能与 `pre_tool_use` 搭配（拦截必须同步判断）
- 每种 Action 类型都有必填字段校验

这些约束在加载阶段就抛出 `HookConfigError`，避免运行时才发现配置错误。

## 设计思考

Hook 系统的设计体现了几个取舍：

**声明式优于编程式**：用户通过配置文件（而非写代码）定义 Hook，降低了使用门槛，也让 Hook 定义可以被序列化和共享。

**同步拦截 vs 异步执行**：`pre_tool_use` 必须是同步的（不能用 `async`），因为工具执行需要等待拦截结果。其他事件可以选择异步执行，不阻塞主循环。

**条件表达式故意简单**：不支持嵌套逻辑、不支持优先级混用。复杂场景拆成多个 Hook，比一个复杂的表达式更易维护。

> 📖 本文是 MewCode 系列的第 11 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
