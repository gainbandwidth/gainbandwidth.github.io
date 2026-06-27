---
title: "MewCode 骨架篇（五）：权限系统"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "五层权限拦截链与规则引擎三级合并"
ShowToc: true
---

## 概述

让 AI Agent 自主操作文件系统、执行 Shell 命令，安全性是第一道门槛。MewCode 设计了一套**五层权限拦截链**：从危险命令硬拦截，到路径沙箱校验，再到 YAML 规则引擎匹配，然后按权限模式兜底，最后才交给用户确认（Human-in-the-Loop）。每一层各司其职，既保证安全，又不至于让用户频繁点击确认。

本文基于 `mewcode/permissions/` 目录下的六个模块展开分析。

## 五层拦截链总览

核心入口是 `PermissionChecker.check()` 方法，它按固定顺序执行五层判定：

```python
def check(self, tool: Tool, arguments: dict) -> Decision:
    content = extract_content(tool.name, arguments)

    # Layer 0: Plan 模式例外放行
    if self.mode == PermissionMode.PLAN:
        if tool.name in _PLAN_MODE_ALLOWED_TOOLS:
            return Decision(effect="allow", reason="Plan mode: allowed tool")

    # Layer 1: 安全的只读命令（自动放行）
    if tool.category == "command" and is_safe_command(content or ""):
        return Decision(effect="allow", reason="Safe read-only command")

    # Layer 1b: 危险命令黑名单
    if tool.category == "command":
        hit, reason = self.detector.detect(content)
        if hit:
            return Decision(effect="deny", reason=f"危险命令拦截: {reason}")

    # Layer 2: 路径沙箱
    if tool.category in ("read", "write") and content:
        ok, reason = self.sandbox.check(content)
        if not ok:
            return Decision(effect="deny", reason=f"路径沙箱拦截: {reason}")

    # Layer 3: 规则引擎
    rule_result = self.rule_engine.evaluate(tool.name, content)
    if rule_result == "allow":
        return Decision(effect="allow", reason="权限规则放行")
    if rule_result == "deny":
        return Decision(effect="deny", reason="权限规则拒绝")

    # Layer 4: 权限模式兜底
    effect = mode_decide(self.mode, tool.category)
    if effect == "allow":
        return Decision(effect="allow", reason=f"权限模式 {self.mode.value} 放行")
    if effect == "deny":
        return Decision(effect="deny", reason=f"权限模式 {self.mode.value} 拒绝")

    # Layer 5: Human-in-the-Loop
    return Decision(effect="ask", reason="需要用户确认")
```

返回的 `Decision` 只有三种 `effect`：`allow`（放行）、`deny`（拒绝）、`ask`（询问用户）。下面逐层拆解。

## Layer 1：危险命令检测

`DangerousCommandDetector` 维护一份正则黑名单，拦截 `rm -rf /`、`mkfs.`、`dd if=...of=/dev/`、fork bomb、`curl | sh` 等高危命令：

```python
_DANGEROUS_PATTERNS = [
    (re.compile(r"rm\s+-[a-z]*r[a-z]*f[a-z]*\s+/\s*$"), "递归强制删除根目录"),
    (re.compile(r"mkfs\."), "格式化磁盘"),
    (re.compile(r":\(\)\{\s*:\|:&\s*\};:"), "fork bomb"),
    (re.compile(r"curl\s+.*\|\s*(ba)?sh"), "管道执行远程脚本"),
    # ...
]
```

与黑名单对称的是一道**白名单**——`is_safe_command()`。`ls`、`cat`、`git status`、`python --version` 等只读命令直接放行，无需用户确认。白名单的判断很保守：一旦命令中出现 `|`、`;`、`&&`、`>`、`$(`、`` ` `` 等拼接符号，立即判定为不安全。

## Layer 2：路径沙箱

`PathSandbox` 解决另一个问题：Agent 不能读写项目目录之外的文件。它在校验时会 `resolve()` 符号链接，防止通过软链逃逸：

```python
class PathSandbox:
    def __init__(self, project_root: str, extra_allowed=None):
        root = Path(project_root).resolve()
        self._allowed_roots = [root, Path(tempfile.gettempdir()).resolve()]

    def check(self, path: str) -> tuple[bool, str]:
        real_path = Path(path).resolve(strict=True)
        for root in self._allowed_roots:
            try:
                real_path.relative_to(root)
                return True, ""
            except ValueError:
                continue
        return False, f"路径 {path} 超出沙箱范围"
```

默认允许项目根目录和系统临时目录，还可以通过 `extra_allowed` 添加额外白名单路径。

## Layer 3：规则引擎三级合并

`RuleEngine` 是权限系统中最灵活的部分。它从三个 YAML 文件加载规则，按**用户级 → 项目级 → 本地级**的顺序合并：

```python
def _load_tiers(self) -> list[list[Rule]]:
    tiers = []
    for p in (self._user_path, self._project_path, self._local_path):
        tiers.append(_load_rules_file(p) if p else [])
    return tiers

def evaluate(self, tool_name: str, content: str) -> Effect | None:
    for rules in self._load_tiers():
        for rule in reversed(rules):  # 同层内后写的规则优先
            if rule.matches(tool_name, content):
                return rule.effect
    return None
```

规则语法是 `ToolName(pattern)` 形式，比如 `Bash(npm install *)` 或 `WriteFile(/etc/*)`。匹配使用 `fnmatch` 通配符。三个层级的优先级设计遵循"就近覆盖"原则：本地级（`.mewcode/permissions.local.yaml`，不提交到 Git）> 项目级 > 用户级。同层内最后写入的规则优先匹配。

## Layer 4：权限模式矩阵

`PermissionMode` 定义了六种权限模式，每种模式对 `read`/`write`/`command` 三类工具有不同的默认行为：

```python
_MODE_MATRIX = {
    PermissionMode.DEFAULT:     {"read": "allow", "write": "ask", "command": "ask"},
    PermissionMode.ACCEPT_EDITS: {"read": "allow", "write": "allow", "command": "ask"},
    PermissionMode.BYPASS:      {"read": "allow", "write": "allow", "command": "allow"},
    PermissionMode.PLAN:        {"read": "allow", "write": "ask",  "command": "ask"},
    # ...
}
```

`DEFAULT` 模式下，读操作自动放行，写操作和命令执行需要确认；`ACCEPT_EDITS` 对写操作也放行；`BYPASS` 则是全开模式（适合 CI 环境）。

## Layer 5：Human-in-the-Loop

当前四层都未做出 allow/deny 决定时，最终返回 `effect="ask"`，由 UI 层弹出确认框让用户决策。这是安全的最后防线——宁可多问一次，也不误操作。

## 设计思考

五层拦截链的顺序很讲究：

1. **安全命令白名单**放在最前面——`ls`、`cat` 这类命令占了 Agent 日常操作的大多数，优先放行能减少 80% 的确认弹窗
2. **危险命令黑名单**紧随其后——即使权限模式是 BYPASS，`rm -rf /` 也不会被执行
3. **路径沙箱**针对文件类工具——防止 Agent 通过绝对路径或 `../` 逃逸项目目录
4. **规则引擎**给用户自定义能力——不同项目可以有不同的安全策略
5. **权限模式**是粗粒度的兜底——一键切换整体安全级别
6. **HITL** 是最终保障——所有未决操作都需要人类确认

这种分层设计的本质是**最小权限原则**的工程化实现：每层都在尝试缩小需要人类决策的范围，只在真正不确定的情况下才打扰用户。

> 📖 本文是 MewCode 系列的第 05 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
