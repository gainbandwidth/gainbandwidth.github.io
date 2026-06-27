---
title: "MewCode 骨架篇（三）：Agent 循环"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "ReAct 循环核心：流式接收、工具分批执行与结果回注"
ShowToc: true
---

## 概述

前两篇介绍了 MewCode 的工具注册与对话管理，本文聚焦最核心的部分——`Agent` 类中的 ReAct 主循环。

ReAct 的核心思想很简单：LLM 推理（Reasoning）→ 执行动作（Action）→ 观察结果（Observation）→ 再推理。MewCode 的 `Agent.run()` 方法将这个思想实现为一个异步流式循环，同时处理了上下文压缩、权限控制、并发执行和错误恢复等工程细节。

## 流式接收与响应收集

循环的起点是 `StreamCollector`，它消费 LLM 返回的异步流，将碎片化的事件拼装成完整的 `LLMResponse`：

```python
class StreamCollector:
    def __init__(self) -> None:
        self.response = LLMResponse()

    async def consume(self, stream):
        async for event in stream:
            if isinstance(event, TextDelta):
                self.response.text += event.text
                yield StreamText(text=event.text)  # 实时转发给 UI
            elif isinstance(event, ToolCallComplete):
                self.response.tool_calls.append(event)
                yield ToolUseEvent(...)
            elif isinstance(event, StreamEnd):
                self.response.stop_reason = event.stop_reason
                self.response.input_tokens = event.input_tokens
```

设计要点：`consume()` 本身是一个 `AsyncIterator`，边收集边 `yield`，UI 层无需等待整个响应完成就能渲染文本流。`LLMResponse` 聚合了 text、tool_calls、thinking_blocks 和 token 用量，是后续所有逻辑的输入。

## 主循环：while True 的 ReAct

`Agent.run()` 的核心是一个 `while True` 循环，每轮迭代对应一次 LLM 调用（一个 turn）：

```python
async def run(self, conversation):
    while True:
        iteration += 1
        if iteration > self.max_iterations:
            yield ErrorEvent(...)
            break

        # 1. 上下文压缩检查
        compact_result = await auto_compact(...)

        # 2. 构建 system prompt + tool schemas
        system = build_system_prompt(...)
        tools = self.registry.get_all_schemas(self.protocol)

        # 3. Layer 1: tool-result budget 裁剪
        api_conv, _ = apply_tool_result_budget(conversation, ...)

        # 4. 流式调用 LLM
        collector = StreamCollector()
        llm_stream = self.client.stream(api_conv, system=system, tools=tools)
        async for event in collector.consume(llm_stream):
            yield event

        # 5. 无 tool_calls → 循环结束
        if not response.tool_calls:
            yield LoopComplete(total_turns=iteration)
            break

        # 6. 执行工具 + 回注结果
        batches = partition_tool_calls(response.tool_calls, self.registry)
        for batch in batches:
            ...
        conversation.add_tool_results_message(tool_results)
        yield TurnComplete(turn=iteration)
```

整个循环可以概括为：**流式接收 → 收集工具调用 → 分批执行 → 结果回注 → 回到循环顶部**。当 LLM 不再请求工具调用时，说明任务完成，循环自然退出。

## 工具分批执行：并发与串行的平衡

`partition_tool_calls()` 将 LLM 一次返回的多个工具调用分成若干 `ToolBatch`，每个 batch 标记为 `concurrent=True` 或 `False`：

```python
def partition_tool_calls(tool_calls, registry):
    batches = []
    for tc in tool_calls:
        tool = registry.get(tc.tool_name)
        safe = tool is not None and tool.is_concurrency_safe
        if safe and batches and batches[-1].concurrent:
            batches[-1].calls.append(tc)  # 合并到上一个并发批
        else:
            batches.append(ToolBatch(concurrent=safe, calls=[tc]))
    return batches
```

只有声明了 `is_concurrency_safe` 的工具（如 `ReadFile`）才会被并发执行；涉及副作用的工具（如 `Bash`、`WriteFile`）严格按顺序执行。并发批通过 `asyncio.gather` 实现：

```python
async def _execute_batch_parallel(self, calls):
    tasks = [self._execute_single_tool_direct(tc) for tc in calls]
    return list(await asyncio.gather(*tasks))
```

## 权限检查：拦截在工具执行之前

`_execute_tool()` 在实际执行工具前，通过 `PermissionChecker` 进行三层判断：

- `deny`：直接拒绝，返回错误
- `ask`：创建 `asyncio.Future`，yield `PermissionRequest` 事件给 UI，等待用户响应
- 允许：继续执行

```python
if decision.effect == "ask":
    future = loop.create_future()
    yield PermissionRequest(tool_name=tc.tool_name, description=desc, future=future)
    response = await future
    if response == PermissionResponse.DENY:
        yield ToolResult(output="Permission denied", is_error=True)
        return
```

用户选择"Always Allow"时，系统会动态添加一条本地规则，后续同类操作自动放行。

## Plan Mode：用权限系统实现只读规划

Plan Mode 并不是一个独立的状态机，而是复用了权限系统的 `PermissionMode.PLAN` 模式。在该模式下：

- 每次 turn 开始前，`build_plan_mode_reminder()` 注入提示，引导 LLM 将计划写入 `.mewcode/plans/` 下的 markdown 文件
- 工具权限受限（本质上是权限策略的效果）
- LLM 调用 `ExitPlanMode` 工具时，主循环检测到后直接跳出

```python
exit_plan_called = any(tc.tool_name == "ExitPlanMode" for tc in response.tool_calls)
if exit_plan_called:
    yield LoopComplete(total_turns=iteration)
    break
```

这种设计的巧妙之处在于：Plan Mode 不需要修改主循环逻辑，只需在权限层注入约束即可。

## run_to_completion：无头执行模式

`run_to_completion()` 是为子 Agent（Sub-agent）场景设计的无交互版本。与 `run()` 的关键差异：

- **无权限弹窗**：遇到 `ask` 决策时，若处于 `DONT_ASK` 模式则自动批准，否则直接返回错误
- **无流式输出**：`collector.consume()` 的返回值被丢弃（`async for _event in ...: pass`），只保留最终文本
- **同步迭代**：用 `for iteration in range(...)` 替代 `while True`，更适合可预期的任务
- **事件回调**：通过 `event_callback` 参数向上层报告进度，而非 yield 事件

```python
async def run_to_completion(self, task, conversation=None, event_callback=None):
    conversation.add_user_message(task)
    for iteration in range(1, self.max_iterations + 1):
        collector = StreamCollector()
        async for _event in collector.consume(llm_stream):
            pass  # 收集但不转发
        if not response.tool_calls:
            break
        # 无交互工具执行
        result = await self._execute_tool_noninteractive(tc)
    return last_text
```

## 设计思考与取舍

**为什么用 `AsyncIterator` 而不是回调？** `yield` 天然支持背压（backpressure），UI 消费不过来时循环会自动暂停，不需要手动管理缓冲区。相比之下，回调模式容易出现事件积压和内存泄漏。

**为什么要分 `run()` 和 `run_to_completion()`？** 交互模式和无人值守模式对权限、错误处理、输出的需求截然不同。与其在一个方法里堆满 `if interactive` 分支，不如拆成两条路径，各自演进。

**连续未知工具计数器（`consecutive_unknown`）的意义：** LLM 有时会"幻觉"出不存在的工具名。偶尔一次可以容忍，但如果连续 3 次调用未知工具，说明模型理解出了系统性偏差，继续执行只会浪费 token，此时主动终止是更负责的选择。

> 📖 本文是 MewCode 系列的第 03 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
