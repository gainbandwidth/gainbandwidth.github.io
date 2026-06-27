---
title: "MewCode 进阶篇（六）：Git Worktree 隔离"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "Git worktree 实现文件级隔离与会话恢复"
ShowToc: true
---

## 概述

当 Agent 在执行代码修改任务时，一个棘手的问题浮现出来：如果同时有多个子 Agent 在修改同一组文件，它们的改动会互相覆盖。传统的做法是让子 Agent 串行工作，但这严重限制了并行能力。

MewCode 的 `worktree/` 模块利用 Git 的 `worktree` 特性实现了文件级隔离——每个子 Agent 在自己的 worktree 分支中独立工作，拥有完整的文件副本，互不干扰。通过 `EnterWorktree` 和 `ExitWorktree` 工具，Agent 可以动态进出隔离环境，并在退出时决定是否保留变更。

## WorktreeManager：核心管理器

`WorktreeManager` 是 worktree 操作的统一入口，管理 worktree 的创建、进入、退出和清理。

### 初始化

```python
class WorktreeManager:
    def __init__(self, repo_root, symlink_directories=None, worktree_dir=None):
        self.repo_root = repo_root
        self.symlink_directories = symlink_directories or []
        self.worktree_dir = worktree_dir or str(
            Path(repo_root) / ".mewcode" / "worktrees"
        )
        self._lock = asyncio.Lock()
        self.active: dict[str, Worktree] = {}
        self.current_session: WorktreeSession | None = None
```

所有 worktree 默认创建在 `.mewcode/worktrees/` 目录下，用 `asyncio.Lock` 保证并发安全。

### 创建 Worktree

`create()` 方法通过 `git worktree add` 创建隔离的工作目录：

```python
async def create(self, name, base_branch="HEAD") -> Worktree:
    async with self._lock:
        err = validate_slug(name)
        if err:
            raise WorktreeError(err)

        flat_slug = flatten_slug(name)
        wt_path = os.path.join(self.worktree_dir, flat_slug)
        branch_name = f"worktree-{flat_slug}"

        # 快速恢复：如果目录已存在，直接复用
        head_sha = self.read_worktree_head_sha(wt_path)
        if head_sha is not None:
            return self._fast_recover(name, wt_path, head_sha)

        result = self._run_git([
            "worktree", "add", "-B", branch_name, wt_path, base_branch,
        ])
        perform_post_creation_setup(self.repo_root, wt_path, ...)
```

创建时有一个重要的**快速恢复**路径：如果目标目录已经存在（可能是上次运行遗留的），直接读取 HEAD SHA 复用，而不是重新 `git worktree add`。

## 后置设置

`setup.py` 中的 `perform_post_creation_setup()` 在 worktree 创建后执行四项配置：

### 1. 复制本地配置文件

```python
LOCAL_CONFIG_FILES = ["settings.local.json", ".env"]

def _copy_local_configs(root, wt):
    for name in LOCAL_CONFIG_FILES:
        src = root / name
        if src.exists():
            shutil.copy2(str(src), str(wt / name))
```

这些文件通常包含 API 密钥和本地设置，不在 Git 跟踪范围内，但子 Agent 运行时需要。

### 2. 同步 Git Hooks

检测主仓库的 Husky 或 `.git/hooks` 目录，在 worktree 中配置相同的 `core.hooksPath`，保证 pre-commit 等钩子正常工作。

### 3. 创建符号链接

这是解决 `node_modules`、`.venv` 等大目录的关键设计：

```python
def _create_symlinks(root, wt, directories):
    for dirname in directories:
        src = root / dirname
        dst = wt / dirname
        if src.exists() and not dst.exists():
            os.symlink(str(src), str(dst))
```

`node_modules` 可能有几十万个文件，完全复制既慢又浪费空间。通过符号链接，所有 worktree 共享同一份依赖目录，既保证了模块解析正确，又避免了重复安装。

### 4. 复制 gitignore 中的文件

通过读取 `.worktreeinclude` 文件中的 glob 模式，选择性复制被 Git 忽略的文件到 worktree 中：

```python
def _copy_ignored_files(root, wt):
    include_file = root / ".worktreeinclude"
    patterns = [line.strip() for line in include_file.read_text()...]
    # 从 git ls-files --others --ignored 中匹配
```

## 进入与退出

### EnterWorktree

`enter()` 方法保存当前工作目录和分支状态，然后切换到 worktree 环境：

```python
async def enter(self, name) -> WorktreeSession:
    wt = self.active.get(name)
    session = WorktreeSession(
        original_cwd=os.getcwd(),
        worktree_path=wt.path,
        worktree_name=name,
        original_branch=self._get_current_branch(),
        original_head_commit=self._get_head_commit(),
    )
    self.current_session = session
    save_worktree_session(self._mewcode_dir, session)
    return session
```

`WorktreeSession` 记录了进入前的完整状态——原始目录、分支名、HEAD commit SHA——这些在退出时用于恢复。

### ExitWorktree

退出时有两个动作选项：

- **`keep`**：保留 worktree 目录和分支，下次可以重新进入
- **`remove`**：删除 worktree，如果 `discard_changes=False` 且有未提交的修改，会抛出错误：

```python
async def exit(self, name, action="keep", discard_changes=False):
    if action == "remove" and not discard_changes:
        changes = count_worktree_changes(wt.path, wt.head_commit)
        if changes.uncommitted > 0 or changes.new_commits > 0:
            raise WorktreeError(
                f"worktree has changes ({changes.uncommitted} uncommitted, "
                f"{changes.new_commits} new commits)."
            )
```

删除操作先执行 `git worktree remove --force`，再执行 `git branch -D` 清理分支。

## 会话持久化与恢复

`session.py` 将会话状态序列化为 JSON 文件：

```python
def save_worktree_session(mewcode_dir, session):
    path = mewcode_dir / "worktree_session.json"
    data = {
        "original_cwd": session.original_cwd,
        "worktree_path": session.worktree_path,
        "worktree_name": session.worktree_name,
        "original_branch": session.original_branch,
        "original_head_commit": session.original_head_commit,
    }
    path.write_text(json.dumps(data, indent=2))
```

`restore_session()` 在 Agent 重启时读取持久化的会话信息。它先验证 worktree 目录是否仍然存在（通过 `read_worktree_head_sha`），如果目录已被删除就清空会话记录。

`read_worktree_head_sha()` 是一个精巧的静态方法——它不调用 `git` 命令，而是直接解析 `.git` 文件、`gitdir` 指向、`HEAD` ref 等文件系统结构，快速读取当前 HEAD commit：

```python
@staticmethod
def read_worktree_head_sha(wt_path) -> str | None:
    git_file = Path(wt_path) / ".git"
    content = git_file.read_text().strip()
    gitdir = Path(content.split(":", 1)[1].strip())
    head_content = (gitdir / "HEAD").read_text().strip()
    if head_content.startswith("ref:"):
        # 解析 ref 指向的实际 commit SHA
```

## 自动清理

`cleanup.py` 实现了过期 worktree 的自动清理机制：

```python
EPHEMERAL_PATTERNS = [
    re.compile(r"^agent-a[0-9a-f]{7}$"),     # 子 Agent 创建的
    re.compile(r"^wf_[0-9a-f]{8}-...$"),      # Workflow 创建的
]

async def cleanup_stale_worktrees(manager, cutoff_hours):
    for entry in worktree_dir.iterdir():
        if not _is_ephemeral(entry.name):
            continue
        if mtime > cutoff:
            continue
        if has_worktree_changes(str(entry), head_sha):
            continue
        if has_unpushed_commits(str(entry)):
            continue
        # 安全删除
```

清理逻辑有三道安全阀：

1. **只清理临时 worktree**：通过正则匹配名称模式，用户手动创建的不会被清理
2. **检查变更**：有未提交修改或新 commit 的不会被清理
3. **检查未推送 commit**：有本地 commit 但未推送到远端的不会被清理

`start_stale_cleanup_task()` 将清理逻辑包装为周期性后台任务，按指定间隔自动运行。

## Slug 校验

`slug.py` 对 worktree 名称做严格校验：

```python
def validate_slug(name: str) -> str | None:
    segments = name.split("/")
    for seg in segments:
        if seg in (".", ".."):
            return "name must not contain '.' or '..'"
        if not _SEGMENT_RE.match(seg):
            return f"invalid segment: {seg!r}"
```

名称只能包含字母、数字、`.`、`-`、`_`，最大长度 64 字符。`flatten_slug()` 将 `/` 替换为 `+`，确保文件系统路径安全。

## 设计思考

**为什么用 Git Worktree 而不是文件系统 copy？** `git worktree` 共享同一个 `.git` 对象库，创建几乎是瞬时的（只复制工作目录文件），而且自带分支管理。相比之下，`cp -r` 既慢又无法追踪分支差异。

**符号链接 vs 复制**：`node_modules` 和 `.venv` 这类目录体积巨大且内容确定性高（由 lock 文件决定），符号链接是最优解。而 `.env` 等配置文件需要独立副本，因为不同 worktree 可能需要不同的配置值。

**安全阀设计**：自动清理宁可多保留也不误删——三道检查（名称模式、变更检测、推送检测）确保不会丢失未保存的工作。

> 📖 本文是 MewCode 系列的第 13 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
