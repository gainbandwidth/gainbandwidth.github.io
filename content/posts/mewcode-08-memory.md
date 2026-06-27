---
title: "MewCode 进阶篇（一）：记忆系统"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "自动记忆提取、Recall 预取与指令文件加载"
ShowToc: true
---

## 概述

LLM 天生是"健忘"的——每轮对话结束后，上下文窗口就是一切。MewCode 的记忆系统分两条线解决这个问题：**自动记忆提取**（`MemoryManager`）在对话过程中异步提取关键信息并持久化到 `memories.md`；**Recall 预取**（`find_relevant_memories`）在每轮对话开始前，用侧查询快速检索相关记忆注入系统提示。此外，**项目指令文件**（`MEWCODE.md`）提供了一套分级的静态指令加载机制。

## 自动记忆提取

`MemoryManager` 的核心思路很直接：每隔 N 轮对话，把最近的聊天记录喂给一个轻量 LLM，让它提取值得长期记忆的信息。

```python
class MemoryManager:
    def __init__(self, project_root: str):
        self._user_path = Path.home() / ".mewcode/memories.md"
        self._project_path = Path(project_root) / ".mewcode/memories.md"
        self._last_extraction_msg_count = 0

    async def extract(self, client, conversation, protocol):
        recent = conversation.history[self._last_extraction_msg_count:]
        if not recent:
            return

        conv_lines = []
        for msg in recent:
            if msg.role == "user":
                conv_lines.append(f"用户: {msg.content}")
            elif msg.role == "assistant":
                conv_lines.append(f"助手: {msg.content}")

        prompt = (
            f"{MEMORY_EXTRACTION_PROMPT}\n\n"
            f"## 当前 memories.md\n{current_memories}\n\n"
            f"## 最近对话\n{chr(10).join(conv_lines)}"
        )
        # 异步流式调用 LLM，收集结果
        async for event in client.stream(extract_conv, system="你是一个记忆提取助手。"):
            if isinstance(event, TextDelta):
                collected += event.text
```

提取的 Prompt 要求 LLM 将信息分为四类：

| 分类 | 作用域 | 示例 |
|------|--------|------|
| 用户偏好 | 用户级 | 偏好简洁代码风格、命名规范 |
| 纠正反馈 | 用户级 | "不要用 var，用 const" |
| 项目知识 | 项目级 | 技术栈、目录结构、部署方式 |
| 参考资料 | 项目级 | 外部文档链接 |

提取完成后，`_write_memories()` 会按分类将内容拆分到用户级（`~/.mewcode/memories.md`）和项目级（`<project>/.mewcode/memories.md`）两个文件。用户级记忆跨项目共享，项目级记忆绑定特定仓库。

## 记忆文件存储与 Frontmatter

更细粒度的记忆以独立 `.md` 文件的形式存储在 `~/.mewcode/memory/` 和项目级 `.mewcode/memory/` 目录下。每个文件带有 YAML frontmatter，包含 `name`、`description`、`type` 三个字段：

```python
VALID_TYPES = {"user", "feedback", "project", "reference"}

def parse_frontmatter(content: str) -> dict[str, str]:
    m = FRONTMATTER_RE.match(content)  # \A---\s*\n(.*?)\n---
    # 提取 name, description, type 三个字段
```

`type` 的四种取值对应上面四类记忆分类，`description` 用于后续 Recall 阶段的语义匹配。扫描时最多读取文件前 30 行来解析 frontmatter，避免大文件的性能问题，总文件数上限 200 个。

## Recall 预取：侧查询 8 秒超时

Recall 的设计是记忆系统中最精巧的部分。在每轮用户消息到达时，系统会异步执行一次"侧查询"（side-query），用一个轻量 LLM 从记忆清单中挑选最相关的记忆文件：

```python
async def find_relevant_memories(query, user_mem_dir, project_mem_dir,
                                  recent_tools, already_surfaced, selector):
    # 1. 扫描两个目录，收集所有记忆文件的 header
    all_headers = []
    if user_mem_dir:
        all_headers.extend(scan_memory_files(user_mem_dir, "user"))
    if project_mem_dir:
        all_headers.extend(scan_memory_files(project_mem_dir, "project"))

    # 2. 过滤已经展示过的记忆
    candidates = [m for m in all_headers if m.file_path not in surfaced]

    # 3. 调用 selector（侧查询 LLM），选出最多 5 条
    selected_filenames = await _select_relevant_memories(query, candidates, ...)

    return [RelevantMemory(path=m.file_path, mtime_ms=m.mtime_ms) for m in ...]
```

Selector 的系统提示非常讲究——它不仅要求 LLM 选择相关记忆，还特别指出"如果最近使用的工具列表中已包含某 API，就不要选择该 API 的使用文档类记忆（因为 Agent 已经在用了），但仍要选择包含告警和已知问题的记忆"。

选出的记忆通过 `render_reminder()` 注入到系统提示中，并附带**新鲜度警告**：超过 1 天的记忆会被标注 "This memory is N days old... Verify against current code before asserting as fact"，防止 Agent 将过时的记忆当作事实断言。

## 项目指令文件：分级加载

与动态记忆不同，`MEWCODE.md` 是用户主动编写的静态指令。`load_instructions()` 按固定顺序加载三个位置的文件：

```python
def load_instructions(project_root: str) -> str:
    paths = [
        root / "MEWCODE.md",              # 项目根目录（提交到 Git）
        root / ".mewcode" / "MEWCODE.md", # 项目 .mewcode 目录
        home / ".mewcode" / "MEWCODE.md", # 用户级全局指令
    ]
    sections = []
    for path in paths:
        if path.exists() and path.is_file():
            content = path.read_text(encoding="utf-8")
            processed = process_includes(content, path.parent, root)
            sections.append(processed)
    return "\n---\n".join(sections)
```

三个文件用 `---` 分隔符拼接后注入系统提示。项目根目录的 `MEWCODE.md` 通常提交到 Git，作为团队共享的项目规范；`.mewcode/MEWCODE.md` 是项目私有的配置；用户级文件则是个人偏好。

指令文件还支持 `@include` 引用，最多递归 5 层深度，且做了路径沙箱校验防止引用项目外的文件：

```python
def process_includes(content, base_dir, project_root, depth=0):
    if depth >= MAX_INCLUDE_DEPTH:
        return content
    for line in lines:
        if stripped.startswith("@include "):
            abs_path = (base_dir / rel_path).resolve()
            abs_path.relative_to(resolved_root)  # 沙箱校验
            included = abs_path.read_text(encoding="utf-8")
            processed = process_includes(included, ..., depth + 1)
```

## 会话持久化：JSONL 格式

`SessionRecord` 将对话历史以 JSONL 格式逐条追加到 `.mewcode/sessions/` 目录下。每条记录包含 `type`（`user`/`assistant`/`tool_result`/`compact_boundary` 等）、`content`、`timestamp` 和可选的 `tool_use_id`。

`compact_boundary` 是一种特殊记录——当上下文压缩（compact）触发时，它内联存储摘要文本和保留的尾部消息，使得后续的 `resume` 可以直接从 boundary 重建压缩后的对话状态，无需重放之前的所有原始消息。

## 设计思考

记忆系统的核心取舍在于**召回精度 vs 上下文成本**。全量注入所有记忆会浪费宝贵的 context window，完全不注入则 Agent 每轮对话都从零开始。MewCode 的方案是用一次轻量 LLM 侧查询做"语义索引"——花少量的 token 成本筛选出最相关的 5 条记忆，性价比远高于 embedding + vector DB 的重型方案。

> 📖 本文是 MewCode 系列的第 08 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
