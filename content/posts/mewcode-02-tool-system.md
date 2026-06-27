---
title: "MewCode 骨架篇（二）：工具系统"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "拆解 MewCode 工具系统的注册、并发执行与结果截断设计"
ShowToc: true
---

## 概述

在 AI Agent 的架构中，工具（Tool）是模型与真实世界之间的桥梁。MewCode 的工具系统围绕三个核心目标展开：**安全可控**（权限检查 + 文件状态追踪）、**高并发**（partition 分批 + 流式预执行）、**长对话友好**（结果落盘 + 渐进式截断）。本文从源码出发，逐层拆解这套系统的设计。

## 一、Tool 抽象：所有工具的公共契约

`mewcode/tools/base.py` 定义了工具系统的核心抽象：

```python
class Tool(ABC):
    name: str
    description: str
    params_model: type[BaseModel]
    category: ToolCategory = "read"       # read | write | command
    is_concurrency_safe: bool = False
    is_system_tool: bool = False
    should_defer: bool = False

    @abstractmethod
    async def execute(self, params: BaseModel) -> ToolResult: ...
```

几个关键字段的设计意图：

- **`category`**：将工具分为 `read`、`write`、`command` 三类，`is_read_only` 属性直接派生自 category，供权限引擎判断是否需要询问用户。
- **`is_concurrency_safe`**：标记工具是否可以安全并发执行。`ReadFile`、`Glob`、`Grep` 等只读工具标记为 `True`，而 `WriteFile`、`Bash` 等写操作工具默认为 `False`。这个标记是后续 `partition_tool_calls` 分批策略的核心依据。
- **`should_defer`**：标记工具是否需要延迟加载。像 `AskUserQuestion`、`EnterWorktree` 等低频工具设置为 `True`，在会话初始不向 LLM 发送 schema，减少 prompt 体积。
- **`params_model`**：使用 Pydantic `BaseModel` 声明参数 schema，自动生成 JSON Schema 并支持多种协议格式（Anthropic / OpenAI）。

`ToolResult` 则非常精简——只有 `output: str` 和 `is_error: bool`，将工具执行结果统一为纯文本 + 状态码的模式。

## 二、25+ 内置工具全景

通过 `create_default_registry()` 和运行时的动态注册，MewCode 注册了 25+ 内置工具，可按功能划分为五组：

| 分组 | 工具 | 特点 |
|------|------|------|
| **文件操作** | ReadFile, WriteFile, EditFile, Glob, Grep | 核心开发工具，集成 FileStateCache |
| **命令执行** | Bash | 异步子进程，支持 timeout（上限 600s） |
| **子 Agent** | AgentTool | 支持前台/后台、worktree 隔离、Team 队友三种模式 |
| **团队协作** | TeamCreate, TeamDelete, TaskCreate/Get/List/Update, SendMessage | 基于 Mailbox 的消息通信 + 共享任务板 |
| **元工具** | AskUserQuestion, LoadSkill, ToolSearch, ExitPlanMode, EnterWorktree, ExitWorktree, SyntheticOutput | 系统级能力，部分标记为 `should_defer` |

以 `EditFile` 为例，它要求 `old_string` 在文件中**恰好出现一次**——零次说明上下文不匹配，多次说明定位不精确——这种"唯一性约束"强制模型提供足够的上下文，避免误编辑。

### FileStateCache：读前写保护

`FileStateCache` 在 `file_state_cache.py` 中实现了一个精巧的"读前写"（read-before-write）门控：

```python
def check(self, path: str) -> tuple[bool, str]:
    entry = self._cache.get(path)
    if entry is None:
        return False, "Error: file has not been read yet."
    _, cached_mtime_ns = entry
    current_mtime_ns = Path(path).stat().st_mtime_ns
    if current_mtime_ns != cached_mtime_ns:
        return False, "Error: file has been modified since last read."
    return True, ""
```

两道闸门：第一道检查文件是否被读过（防止盲写）；第二道检查文件的 `mtime_ns` 是否变化（防止写覆盖外部修改）。这个设计在 Agent 长时间运行的场景下尤为重要——Agent 可能在 20 轮迭代前读过某个文件，期间另一个子 Agent 修改了它，这道门控能避免冲突写入。

## 三、partition_tool_calls：并发分批执行

当 LLM 在一次响应中返回多个 tool call 时，MewCode 不会简单地串行执行，也不会粗暴地全部并发，而是通过 `partition_tool_calls` 进行智能分批：

```python
def partition_tool_calls(
    tool_calls: list[ToolCallComplete],
    registry: ToolRegistry,
) -> list[ToolBatch]:
    batches: list[ToolBatch] = []
    for tc in tool_calls:
        tool = registry.get(tc.tool_name)
        safe = tool is not None and tool.is_concurrency_safe \
               and registry.is_enabled(tc.tool_name)
        if safe and batches and batches[-1].concurrent:
            batches[-1].calls.append(tc)
        else:
            batches.append(ToolBatch(concurrent=safe, calls=[tc]))
    return batches
```

算法逻辑很直观：遍历 tool call 列表，如果当前调用是 `concurrency_safe` 的，就尝试合并到上一个 concurrent 批次；否则开一个新的串行批次。效果是：**连续的只读调用（如同时 Grep + Glob + ReadFile）会被合并为一个并发批次**，而写操作则单独串行执行。

并发批次通过 `_execute_batch_parallel` 使用 `asyncio.gather` 同时执行：

```python
async def _execute_batch_parallel(self, calls):
    tasks = [self._execute_single_tool_direct(tc) for tc in calls]
    return list(await asyncio.gather(*tasks))
```

这个设计在模型要求"先读 5 个文件"的场景下能将耗时从 5x 降到接近 1x。

## 四、流式预执行：StreamingExecutor

MewCode 不等待 LLM 完整输出所有 token 后才开始执行工具。`StreamingExecutor` 允许在 LLM 还在输出参数 JSON 的过程中就提前提交工具执行：

```python
class StreamingExecutor:
    def __init__(self) -> None:
        self._tasks: list[tuple[int, asyncio.Task]] = []
        self._order = 0

    def submit(self, coro) -> None:
        task = asyncio.create_task(coro)
        self._tasks.append((self._order, task))
        self._order += 1

    async def collect_results(self) -> list[_ToolExecResult]:
        tasks = [t for _, t in sorted(self._tasks, key=lambda x: x[0])]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        ...
```

每个 tool call 在 stream 解析完参数后立即 `submit` 为 `asyncio.Task`，最终通过 `collect_results` 按提交顺序收集结果。这意味着当 LLM 花 3 秒输出 tool call 的 JSON 时，一个 Bash 命令可能已经提前执行了 2 秒，**实际端到端延迟减少了重叠部分**。

## 五、工具结果持久化与三级截断

长对话中，工具输出可能累积到数十万字符，挤占 LLM 的 context window。MewCode 设计了三级截断 + 落盘机制：

**第一级：单条超限（`SINGLE_RESULT_CHAR_LIMIT = 50,000`）**

```python
def _maybe_persist_or_truncate(self, tool_use_id, text):
    if len(text) > SINGLE_RESULT_CHAR_LIMIT:
        fp = persist_tool_result(tool_use_id, text, self.session_dir)
        return make_persisted_preview(text, fp)
    if len(text) > MAX_OUTPUT_CHARS:
        return text[:MAX_OUTPUT_CHARS] + "\n… (output truncated)"
    return text
```

超过 50K 字符的单条结果会落盘到 `.mewcode/sessions/{tool_use_id}.txt`，对话中只保留 2KB 预览 + 文件路径。模型需要时可以用 `ReadFile` 重新读取完整内容。

**第二级：聚合超限（`AGGREGATE_CHAR_LIMIT = 200,000`）**

`apply_tool_result_budget` 在每轮 LLM 调用前检查所有 tool result 的总字符数。如果聚合超限，按大小降序排列，逐个落盘直到总量回到阈值以下。

**第三级：陈旧裁剪（`_snip_stale_messages`）**

超过 10 轮（`KEEP_RECENT_TURNS`）之前的旧工具结果，内容被截断为 2K 字符并标记为 `<snipped>`。这保证远期历史不会占用过多 token 预算。

这三道防线配合 `auto_compact`（Layer 2 压缩），使得 MewCode 能够在数百轮对话中维持稳定的上下文消耗。

## 六、延迟加载：Deferred Tools

当注册的工具数量较多时（加上 skill 动态注册的工具可能达到 30+），将所有 schema 放入 system prompt 会浪费大量 token。MewCode 的 `ToolRegistry` 通过 `should_defer` 标记实现了按需加载：

- 标记为 `should_defer = True` 的工具在初始时不发送 schema，只以名字列表的形式告知模型"这些工具可用"。
- 模型通过 `ToolSearch` 工具的 `select:<name>` 语法加载特定工具的 schema，或按关键词搜索相关工具。
- `ToolRegistry.search_deferred` 实现了简单的评分搜索——匹配工具名得 10 分，匹配描述得 5 分，部分匹配再按词加分。

这个设计让 MewCode 在工具数量扩展时不会线性增加 prompt 开销。

## 设计思考与取舍

1. **并发的安全边界**：`is_concurrency_safe` 标记由工具开发者手动设置，而非自动推断。这是一个务实的选择——并发安全性取决于工具的具体行为（比如 `Bash` 理论上可以并发，但两条 shell 命令可能有副作用依赖），无法通过静态分析可靠判断。

2. **截断而非丢弃**：大结果落盘后仍保留文件路径和预览，模型可以随时通过 `ReadFile` 找回。这比直接丢弃更友好，也比压缩（如 gzip）更简单——文本文件的压缩率有限，但落盘的"零成本"访问模式对 Agent 更直觉。

3. **流式预执行的时序风险**：在 LLM 还没输出完参数时就启动工具执行，意味着如果参数 JSON 解析出错，工具可能拿到不完整的数据。MewCode 的处理方式是只在参数完整解析后才 submit，用解析延迟换取执行安全。

4. **FileStateCache 的局限性**：`mtime_ns` 只能检测文件级别的变更，无法检测"内容相同但 mtime 不同"的情况（例如 `touch` 操作）。但在实际使用中，这种 false positive 只会导致模型重新读一次文件，代价可接受。

> 📖 本文是 MewCode 系列的第 02 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
