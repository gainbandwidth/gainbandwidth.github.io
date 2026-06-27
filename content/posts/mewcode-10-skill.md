---
title: "MewCode 进阶篇（三）：Skill 技能包"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "Markdown 驱动的技能包设计与 SOP 注入机制"
ShowToc: true
---

## 概述

MewCode 的 Skill 系统让 Agent 拥有了"可编程的工作流"。与硬编码的工具不同，Skill 以 Markdown 文件（`SKILL.md`）定义，包含 YAML frontmatter 元数据和 SOP（标准操作流程）正文。Agent 通过 `LoadSkill` 工具按需激活 Skill，将其 SOP 注入系统提示，还可以注册 Skill 自带的自定义工具。整个过程围绕四个模块展开：`parser`（解析）、`loader`（加载）、`executor`（执行）和 `directory`（自定义工具注册）。

## SKILL.md 格式

每个 Skill 是一个 Markdown 文件，结构如下：

```yaml
---
name: code-review
description: 对代码变更进行结构化审查
mode: fork
context: recent
allowedTools:
  - ReadFile
  - Bash
---

## 步骤
1. 读取变更的文件列表
2. 逐文件审查，关注以下维度...
3. 输出审查报告

## 注意事项
- 不要修改代码，只输出建议
```

`parser.py` 中的 `parse_skill_file()` 负责解析这个格式：

```python
def parse_skill_file(path: Path) -> SkillDef:
    raw = path.read_text(encoding="utf-8")
    meta, body = parse_frontmatter(raw)
    _validate_meta(meta, str(path))
    return SkillDef(
        name=meta["name"],
        description=meta["description"],
        prompt_body=body,
        allowed_tools=meta.get("allowedTools", []),
        mode=meta.get("mode", "inline"),
        context=meta.get("context", "full"),
    )
```

`SkillDef` 数据类包含了 Skill 的全部元信息：

| 字段 | 含义 | 可选值 |
|------|------|--------|
| `name` | 技能名称 | 小写字母 + 数字 + 连字符 |
| `description` | 一行描述 | 用于目录匹配 |
| `mode` | 执行模式 | `inline`（注入主 Agent）/ `fork`（独立子 Agent） |
| `context` | 上下文传递 | `full` / `recent` / `none` |
| `allowedTools` | 工具白名单 | fork 模式下限制可用工具 |

## 三层加载优先级

`SkillLoader` 从三个来源加载 Skill，优先级从高到低：

```python
class SkillLoader:
    def load_all(self) -> dict[str, SkillDef]:
        seen = {}
        # 1. 项目级（.mewcode/skills/）
        for skill in self._scan_directory(self._project_dir, "project"):
            if skill.name not in seen:
                seen[skill.name] = skill
        # 2. 用户级（~/.mewcode/skills/）
        for skill in self._scan_directory(self._user_dir, "user"):
            if skill.name not in seen:
                seen[skill.name] = skill
        # 3. 内置（mewcode.skills.builtins 包）
        for skill in self._load_builtins():
            if skill.name not in seen:
                seen[skill.name] = skill
```

扫描每个目录时，支持两种文件布局：单文件 `name.md` 和目录 `name/SKILL.md`（后者可以携带 `references/` 子目录存放自定义工具实现）。

内置 Skill 通过 `importlib.resources` 从 Python 包内加载，无需用户手动安装。

## 热重载

`SkillLoader.get()` 在返回 Skill 时会检查源文件是否已更新，实现热重载：

```python
def get(self, name: str) -> SkillDef | None:
    skill = self._skills.get(name)
    if skill.source_path is not None:
        try:
            fresh = parse_skill_file(skill.source_path)
            self._skills[name] = fresh
            self._cache[name] = fresh
            return fresh
        except SkillParseError as e:
            log.warning("Hot-reload failed, using cached version")
            return self._cache.get(name, skill)
```

这意味着用户编辑 SKILL.md 后无需重启 MewCode，下次调用时自动使用最新版本。如果解析失败，则回退到缓存版本，不会因为一个语法错误导致 Skill 不可用。

## LoadSkill：SOP 注入系统提示

`LoadSkill` 是一个注册到 Agent 工具列表中的特殊工具，LLM 在判断需要使用某个 Skill 时会主动调用它：

```python
class LoadSkill(Tool):
    name = "LoadSkill"
    description = "Load and activate a skill by name. "
                  "The skill's SOP will be pinned to the environment context."

    async def execute(self, params: BaseModel) -> ToolResult:
        skill = self._loader.get(params.name)
        self._agent.activate_skill(skill.name, skill.prompt_body)

        # 注册 Skill 自带的自定义工具
        if skill.is_directory and skill.source_path is not None:
            tool_count = register_skill_tools(skill_dir, self._agent.registry)

        return ToolResult(output=f"Skill '{skill.name}' activated. SOP pinned.")
```

`activate_skill()` 将 Skill 的 `prompt_body`（Markdown 正文）追加到 Agent 的系统提示中，成为后续对话的指导原则。这就是"SOP 注入"——用 Markdown 编写的标准操作流程直接引导 Agent 的行为。

## Skill 目录匹配

在 MewCode 启动时，`SkillLoader` 的 `get_catalog()` 方法会生成一份 Skill 目录清单，注入到系统提示中：

```python
catalog = self.skill_loader.get_catalog()
if catalog:
    lines = ["You can use the following Skills:", ""]
    for name, desc in catalog:
        lines.append(f"- {name}: {desc}")
    lines.append("If the user's request matches a Skill, call LoadSkill to activate it.")
    self.agent.set_skill_catalog("\n".join(lines))
```

Agent 看到目录后，会根据用户请求与描述进行语义匹配，自动决定是否调用 `LoadSkill`。这比硬编码 if-else 灵活得多——新增一个 Skill 只需放一个 Markdown 文件，无需修改任何代码。

## Fork 模式与工具隔离

`SkillExecutor` 支持两种执行方式。`inline` 模式直接将 SOP 注入主 Agent，共享上下文和工具。`fork` 模式则创建一个独立的子 Agent：

```python
async def execute_fork(self, skill: SkillDef, args: str) -> str:
    fork_conv = ConversationManager()
    context_messages = self._build_fork_context(skill.context)

    filtered_registry = filter_tool_registry(
        self.agent.registry, skill.allowed_tools
    )
    fork_agent = Agent(
        client=self.client,
        registry=filtered_registry,
        permission_checker=None,  # fork agent 无需权限确认
        context_window=self.agent.context_window,
    )
    async for event in fork_agent.run(fork_conv):
        if isinstance(event, StreamText):
            result_parts.append(event.text)
```

Fork 模式的两个关键设计：一是通过 `filter_tool_registry` 限制子 Agent 可用的工具（只保留 `allowedTools` 中声明的），实现最小权限；二是 `permission_checker=None`——子 Agent 的操作不需要再次确认用户，因为它的能力范围已经被白名单约束。

上下文传递支持三种策略：`full`（完整历史摘要）、`recent`（最近 5 条消息）、`none`（空白开始）。

## 自定义工具注册

Skill 目录下的 `tool.json` 可以定义自定义工具，`references/` 子目录中放置对应的 Python 实现：

```python
def register_skill_tools(skill_dir: Path, registry: ToolRegistry) -> int:
    schemas = parse_tool_json(skill_dir / "tool.json")
    references_dir = skill_dir / "references"
    for schema in schemas:
        impl = load_tool_implementation(references_dir, tool_name)
        tool = SkillCustomTool(tool_name, description, schema, impl)
        registry.register(tool)
```

`load_tool_implementation()` 通过 `importlib` 动态加载 Python 模块，查找其中的 `execute()` 函数。这使得 Skill 可以打包复杂的领域逻辑，而不只是 Markdown 提示。

## 设计思考

Skill 系统的设计核心是**Markdown 即协议**。相比传统的代码插件，Markdown 格式的 Skill 有两个优势：一是 LLM 天然擅长理解和遵循 Markdown 中的指令，SOP 注入后几乎零适配成本；二是非开发者也能编写和维护 Skill，降低了扩展门槛。`inline` 与 `fork` 两种模式覆盖了从简单提示注入到独立子 Agent 的全光谱需求，`allowedTools` 白名单则保证了 fork 模式下的安全边界。

> 📖 本文是 MewCode 系列的第 10 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
