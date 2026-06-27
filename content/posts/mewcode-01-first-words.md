---
title: "MewCode 骨架篇（一）：让 LLM 开口说话"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "统一三家流式协议，建立对话管理骨架"
ShowToc: true
---

## 概述

一个 Agent 框架最核心的能力，归根结底是"让 LLM 开口说话，再把话听明白"。MewCode 的对话层由三个模块协作完成：`client.py` 负责调用各家 API 并统一流式输出、`conversation.py` 管理消息历史与 token 估算、`serialization.py` 将内部消息适配成各协议的请求格式。本文拆解这三层的设计脉络。

## 一、StreamEvent：一套接口吃掉三套协议

Anthropic Messages API、OpenAI Responses API、OpenAI Chat Completions API——三家的 SSE 事件结构完全不同，但上层调度器不应该关心这些差异。MewCode 的解法是定义一个 **tagged union**：

```python
StreamEvent = (
    TextDelta | ThinkingDelta | ThinkingComplete
    | ToolCallStart | ToolCallDelta | ToolCallComplete
    | StreamEnd
)
```

每种客户端的 `stream()` 方法都是一个 `AsyncIterator[StreamEvent]`，内部各自做协议翻译：

- **AnthropicClient** 解析 `content_block_start` / `content_block_delta` / `content_block_stop` 事件链，把 `text_delta` 映射为 `TextDelta`，把 `input_json_delta` 累积后在 block 结束时发射 `ToolCallComplete`。
- **OpenAIClient**（Responses API）监听 `response.output_text.delta` 和 `response.function_call_arguments.done`，后者需要先从 JSON 字符串解析出 `arguments`。
- **OpenAICompatClient**（Chat Completions）面对更碎片的 `tool_calls` 索引式 delta，用 `active_calls: dict[int, dict]` 按索引跟踪每个进行中的工具调用，在 `finish_reason == "tool_calls"` 时批量完成。

上层调度器只需要 `async for event in client.stream(...)` 然后做 `isinstance` 分派，完全不需要知道底层是哪家 API。这是经典的 **Adapter Pattern** 在流式场景下的应用。

## 二、工厂函数与错误体系

`create_client(config)` 是入口：

```python
def create_client(config: ProviderConfig) -> LLMClient:
    if config.protocol == "anthropic":
        return AnthropicClient(config)
    elif config.protocol == "openai":
        return OpenAIClient(config)
    elif config.protocol == "openai-compat":
        return OpenAICompatClient(config)
```

三个客户端共享一套错误层级——`LLMError` 是基类，`AuthenticationError`、`RateLimitError`、`NetworkError` 各司其职。特别值得注意的是 `RateLimitError` 携带了 `retry_after` 字段，方便上层实现退避重试。每个客户端的 `stream()` 方法内部都做了异常捕获与重映射，绝不让 SDK 原生异常泄漏到框架层。

## 三、Token 锚点追踪：精确与估算的混合策略

LLM 的 context window 有限，Agent 需要随时知道"对话还剩多少空间"。但逐条消息做 tokenizer 计算开销太大，而 API 每次响应会返回真实的 token 用量。MewCode 的策略是**锚点 + 尾部估算**：

```python
def record_usage_anchor(self, input_tokens, output_tokens=0,
                        cache_read=0, cache_creation=0):
    self.baseline_tokens = (
        input_tokens + cache_read + cache_creation + output_tokens
    )
    self.anchor_count = len(self.history)
```

每次收到 `StreamEnd` 事件，调度器调用 `record_usage_anchor()` 钉下一个"真实用量锚点"。之后调用 `current_tokens()` 时：

- **锚点以内的消息**：直接信任 `baseline_tokens`（API 给的精确值）。
- **锚点之后新追加的消息**：用 `_CHARS_PER_TOKEN = 3.5` 做粗略字符估算。

这个混合策略的好处是：刚收到 API 响应后估算几乎精确，随着新消息追加误差逐渐增大，但下一次 API 响应到来后锚点又会被刷新。如果经历了上下文压缩（`replace_history()`），锚点会被清零，退化为全量字符估算，直到下一次 API 调用重建锚点。

## 四、消息序列化：三层适配器

`ConversationManager` 内部只存储协议无关的 `Message` 对象（包含 `content`、`tool_uses`、`tool_results`、`thinking_blocks`）。发送请求前，由 `serialization.py` 中的三个 builder 函数负责转换：

| 函数 | 目标协议 | 关键差异 |
|------|---------|---------|
| `build_anthropic_messages()` | Anthropic Messages | 内容块数组，`thinking`/`tool_use`/`tool_result` 并列 |
| `build_openai_input()` | OpenAI Responses | `function_call` + `function_call_output` 扁平结构 |
| `build_chat_completion_messages()` | Chat Completions | `tool_calls` 嵌套在 assistant 下，工具结果用 `role: "tool"` |

一个容易忽略的细节：`build_anthropic_messages()` 会**合并连续的同角色纯文本消息**——因为 Anthropic 协议要求 user/assistant 严格交替出现，而 MewCode 的 system-reminder 注入机制可能产生连续两条 user 消息。序列化层默默把这个问题消除了。

## 五、Prompt Cache：省钱的隐藏逻辑

Anthropic 客户端还暗含了 prompt cache 优化。`_mark_last_user_tail_for_cache()` 在最后一条 user 消息的尾部打上 `cache_control: {"type": "ephemeral"}` 标记，`_mark_last_tool_for_cache()` 则在 tools 列表尾部做同样的事。Anthropic 会缓存到这些断点为止的前缀，命中缓存的 token 只需 10% 的费用。

对于 tools 列表，代码特意做了**浅拷贝**而非原地修改，因为 tool schema 通常是注册表里的模块级单例，直接修改会污染全局状态。

`StreamEnd` 事件中携带了 `cache_read` 和 `cache_creation` 字段，让上层能统计缓存命中率。OpenAI 侧只暴露 `cached_tokens`（对应 `cache_read`），`cache_creation` 始终为 0——这个差异在 `StreamEnd` 的注释中有明确说明。

## 设计思考

1. **为什么不直接用 LangChain / LiteLLM？** 这些库的抽象层过重，且对流式 thinking（extended thinking）的支持不够精细。MewCode 自己写适配层，代价是约 500 行代码，换来的是对 `ThinkingBlock`、`signature` 等细节的完全掌控。

2. **锚点策略 vs 实时 tokenizer**：实时 tokenize 每条消息的精度提升不到 5%，但每次估算都需要加载 tokenizer 模型。对于 Agent 场景，粗略估算已经足够——真正做 context window 压缩时，框架会直接调用 API 让 LLM 自己摘要，而非依赖精确到个位数的 token 计数。

3. **三种协议并存**看似冗余，实则是务实选择：`anthropic` 提供最完整的 thinking + caching；`openai`（Responses API）是 OpenAI 最新的接口；`openai-compat`（Chat Completions）则覆盖 vLLM、Ollama 等开源部署场景。

> 📖 本文是 MewCode 系列的第 01 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
