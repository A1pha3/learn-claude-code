# s12：工作树与任务隔离 —— 完美的终点

> **核心口号**：*"隔离靠目录，协调靠任务 ID"*

---

## 学习目标

完成本章后，你将能够：

1. **理解隔离的价值** —— 为什么共享目录在并行任务中会产生冲突
2. **掌握 Git Worktree** —— 用 Git 原生能力创建隔离的工作目录
3. **实现任务-工作树绑定** —— 任务（控制面）与工作树（执行面）的关联
4. **设计事件流** —— 通过 append-only 日志追踪完整生命周期

**学习分层表**：

| 层级 | 目标 | 检验标准 |
|------|------|----------|
| ⭐ 基础 | 理解隔离的必要性与 Worktree 概念 | 能解释共享目录冲突的根因，能描述 create -> run -> remove 基本流程 |
| ⭐⭐ 进阶 | 独立实现任务-工作树绑定工作流 | 能完成完整闭环：创建任务 -> 绑定工作树 -> 隔离执行 -> remove/keep，理解 EventBus 审计价值 |
| ⭐⭐⭐ 专家 | 设计并行执行与事件查询系统 | 能实现 `run_parallel` 多工作树并行、事件按类型/时间过滤，能评估非 Git 降级方案 |

---

## 0. 上手演练（建议先做）

先用 3 分钟跑一个最小流程，再进入正文会更容易理解：

```bash
uv run python agents/s12_worktree_task_isolation.py
```

建议观察 3 件事：

1. 任务创建后，何时与工作树发生绑定。
2. 在不同工作树执行同类命令时，输出如何保持隔离。
3. 移除工作树时，任务状态和事件流如何同步变化。

### 本章完成标准

- 能解释"共享目录冲突"与"工作树隔离"在协作稳定性上的差异。
- 能独立完成"创建工作树 -> 绑定任务 -> 执行命令 -> 保留/移除回收"流程。
- 能说清事件流在审计与故障恢复中的价值。

---

## 1. 问题的本质

### 1.1 共享目录的问题

在 s11 中，多个队友自主认领任务，但所有人在同一个目录下工作：

```
Alice (任务 #1: 修改 auth.py):  write_file("auth.py", modified_content)
Bob   (任务 #2: 修改 auth.py):  write_file("auth.py", other_content)
结果: Bob 的修改覆盖了 Alice 的修改！
```

**四个根因**：
1. **文件冲突** —— 多人同时修改同一文件
2. **状态污染** —— 未提交的更改混在一起
3. **难以回滚** —— 无法单独撤销某个任务的修改
4. **测试干扰** —— 一个任务的测试影响另一个任务

### 1.2 理想的隔离

```
主仓库: /workspace/main/
├── .git/
├── auth.py
└── api.py

工作树 1: /workspace/.worktrees/auth-refactor/
├── auth.py (Alice 在这里修改)
└── api.py

工作树 2: /workspace/.worktrees/add-testing/
├── auth.py (原样，不受 Alice 影响)
├── api.py (Bob 在这里修改)
└── tests/

Alice 和 Bob 在完全独立的目录中工作，互不干扰！
```

核心思想：**任务（控制面）管理工作目标，工作树（执行面）管理实际操作。通过 task_id 将两者绑定。**

---

## 2. Git Worktree 基础

### 2.1 什么是 Worktree

Git Worktree 允许在同一仓库下检出多个工作目录，每个目录有独立的分支和文件状态：

```bash
# 创建新的 worktree（基于 HEAD 创建分支 wt/auth-refactor）
git worktree add .worktrees/auth-refactor -b wt/auth-refactor HEAD

# 目录结构：
# main/           <-- 主工作树
# .worktrees/auth-refactor/  <-- 新工作树
#   └── .git 文件指向 main/.git/worktrees/auth-refactor/

# 列出所有 worktree
git worktree list

# 删除 worktree
git worktree remove .worktrees/auth-refactor

# 强制删除（有未提交更改时）
git worktree remove --force .worktrees/auth-refactor
```

### 2.2 Worktree 的优势

| 特性 | 说明 |
|------|------|
| **隔离** | 完全独立的文件副本，互不干扰 |
| **共享历史** | 共享 .git 目录，节省磁盘空间 |
| **独立分支** | 每个 worktree 有自己的分支 |
| **原子操作** | Git 原生支持，安全可靠 |
| **快速创建** | 硬链接方式，创建速度极快 |

### 2.3 仓库根目录检测

源码中使用 `detect_repo_root` 函数自动定位仓库根目录：

```python
def detect_repo_root(cwd: Path) -> Path | None:
    """通过 git rev-parse --show-toplevel 检测仓库根目录"""
    try:
        r = subprocess.run(
            ["git", "rev-parse", "--show-toplevel"],
            cwd=cwd,
            capture_output=True,
            text=True,
            timeout=10,
        )
        if r.returncode != 0:
            return None
        root = Path(r.stdout.strip())
        return root if root.exists() else None
    except Exception:
        return None

REPO_ROOT = detect_repo_root(WORKDIR) or WORKDIR
```

如果不在 Git 仓库中，回退到当前工作目录。后续 `.tasks` 和 `.worktrees` 目录都基于 `REPO_ROOT` 创建。

---

## 3. 三大核心组件

s12 的架构由三个核心类组成：

```
TaskManager     ── 控制面：任务生命周期管理
EventBus        ── 观测面：append-only 事件日志
WorktreeManager ── 执行面：Git Worktree 的创建、执行、销毁
```

### 3.1 EventBus：事件总线

EventBus 是 append-only 事件日志，记录所有生命周期事件，提供可观测性。

```python
class EventBus:
    def __init__(self, event_log_path: Path):
        self.path = event_log_path
        self.path.parent.mkdir(parents=True, exist_ok=True)
        if not self.path.exists():
            self.path.write_text("")

    def emit(
        self,
        event: str,
        task: dict | None = None,
        worktree: dict | None = None,
        error: str | None = None,
    ):
        payload = {
            "event": event,
            "ts": time.time(),
            "task": task or {},
            "worktree": worktree or {},
        }
        if error:
            payload["error"] = error
        with self.path.open("a", encoding="utf-8") as f:
            f.write(json.dumps(payload) + "\n")

    def list_recent(self, limit: int = 20) -> str:
        # 读取最后 N 条事件（最多 200 条）
        ...
```

**事件类型一览**（从源码中实际使用的全部事件）：

| 事件名 | 触发时机 | 携带数据 |
|--------|---------|---------|
| `worktree.create.before` | 创建工作树之前 | task.id, worktree.name, worktree.base_ref |
| `worktree.create.after` | 创建工作树成功之后 | task.id, worktree.name/path/branch/status |
| `worktree.create.failed` | 创建工作树失败 | task.id, worktree.name, error |
| `worktree.remove.before` | 移除工作树之前 | task.id, worktree.name/path |
| `worktree.remove.after` | 移除工作树成功之后 | task.id, worktree.name/path/status |
| `worktree.remove.failed` | 移除工作树失败 | task.id, worktree.name, error |
| `worktree.keep` | 标记保留工作树时 | task.id, worktree.name/path/status |
| `task.completed` | 移除工作树时完成任务 | task.id/subject/status, worktree.name |

事件存储在 `.worktrees/events.jsonl` 文件中，每行一个 JSON 对象。

### 3.2 TaskManager：任务管理器

管理任务的生命周期，每个任务存储为 `.tasks/task_{id}.json` 文件。

```python
class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(parents=True, exist_ok=True)
        self._next_id = self._max_id() + 1
```

**任务数据结构**：

```json
{
  "id": 1,
  "subject": "实现认证重构",
  "description": "",
  "status": "pending",
  "owner": "",
  "worktree": "",
  "blockedBy": [],
  "created_at": 1730000000.0,
  "updated_at": 1730000000.0
}
```

**关键方法**：

| 方法 | 功能 |
|------|------|
| `create(subject, description)` | 创建新任务，status 默认 "pending" |
| `get(task_id)` | 获取任务详情 |
| `update(task_id, status, owner)` | 更新任务状态或 owner |
| `bind_worktree(task_id, worktree, owner)` | 绑定任务到工作树，pending 自动转为 in_progress |
| `unbind_worktree(task_id)` | 解除工作树绑定（将 worktree 置空） |
| `list_all()` | 列出所有任务 |

`bind_worktree` 是绑定逻辑的关键：将 `task["worktree"]` 设为工作树名称，若任务状态为 `pending` 则自动转为 `in_progress`。

### 3.3 WorktreeManager：工作树管理器

本章核心类，负责 Git Worktree 的创建、执行和销毁。

```python
class WorktreeManager:
    def __init__(self, repo_root: Path, tasks: TaskManager, events: EventBus):
        self.repo_root = repo_root
        self.tasks = tasks
        self.events = events
        self.dir = repo_root / ".worktrees"
        self.dir.mkdir(parents=True, exist_ok=True)
        self.index_path = self.dir / "index.json"
        # 初始化索引
        if not self.index_path.exists():
            self.index_path.write_text(json.dumps({"worktrees": []}, indent=2))
        self.git_available = self._is_git_repo()
```

**索引数据结构**（存储在 `.worktrees/index.json`）：

```json
{
  "worktrees": [
    {
      "name": "auth-refactor",
      "path": "/path/to/.worktrees/auth-refactor",
      "branch": "wt/auth-refactor",
      "task_id": 1,
      "status": "active",
      "created_at": 1730000000.0
    }
  ]
}
```

注意：索引中的 `worktrees` 是一个**数组**（不是字典），通过 `_find(name)` 方法按 name 查找。

#### 3.3.1 名称校验

```python
def _validate_name(self, name: str):
    if not re.fullmatch(r"[A-Za-z0-9._-]{1,40}", name or ""):
        raise ValueError(
            "Invalid worktree name. Use 1-40 chars: letters, numbers, ., _, -"
        )
```

工作树名称只允许 ASCII 字符（字母、数字、点、下划线、连字符），长度 1-40 字符。名称会直接用作目录名和分支名（`wt/{name}`），因此不支持中文、空格、斜杠等：

```
# 以下会触发 ValueError：
"认证重构"       # 中文字符
"task #1"       # 空格和 #
"my worktree"   # 空格
"tree/v1"       # 斜杠

# 以下合法：
"auth-refactor"  # 字母 + 连字符
"task_1"         # 字母 + 下划线 + 数字
"v2.0.fix"       # 字母 + 点 + 数字
```

这一限制源于：正则 `[A-Za-z0-9._-]` 只匹配 ASCII 子集；名称直接用作目录名和分支名（`wt/{name}`），特殊字符可能导致 Git 命令失败。

#### 3.3.2 创建工作树

```python
def create(self, name: str, task_id: int = None, base_ref: str = "HEAD") -> str:
    self._validate_name(name)
    # 检查重名
    if self._find(name):
        raise ValueError(f"Worktree '{name}' already exists in index")
    # 检查任务是否存在
    if task_id is not None and not self.tasks.exists(task_id):
        raise ValueError(f"Task {task_id} not found")

    path = self.dir / name
    branch = f"wt/{name}"

    # 发出 before 事件
    self.events.emit("worktree.create.before", ...)

    # 执行 git worktree add
    self._run_git(["worktree", "add", "-b", branch, str(path), base_ref])

    # 写入索引
    entry = { "name": name, "path": str(path), "branch": branch,
              "task_id": task_id, "status": "active", "created_at": time.time() }

    # 绑定任务（如果提供了 task_id）
    if task_id is not None:
        self.tasks.bind_worktree(task_id, name)

    # 发出 after 事件
    self.events.emit("worktree.create.after", ...)
```

关键实现细节：
- 分支名自动生成为 `wt/{name}`
- `base_ref` 参数默认为 `"HEAD"`，可以指定其他 Git 引用（如某个 commit hash 或 tag）
- 创建失败时发出 `worktree.create.failed` 事件并向上抛出异常

#### 3.3.3 在工作树中执行命令

```python
def run(self, name: str, command: str) -> str:
    # 危险命令拦截
    dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
    if any(d in command for d in dangerous):
        return "Error: Dangerous command blocked"

    wt = self._find(name)
    path = Path(wt["path"])

    r = subprocess.run(
        command,
        shell=True,          # 使用 shell 执行
        cwd=path,            # 关键：在工作树目录中执行
        capture_output=True,
        text=True,
        timeout=300,         # 5 分钟超时
    )
    out = (r.stdout + r.stderr).strip()
    return out[:50000] if out else "(no output)"
```

关键实现细节：
- `shell=True` 支持管道等 shell 特性
- `cwd=path` 确保命令在目标工作树目录中运行
- 超时 300 秒，输出截断到 50000 字符
- 危险命令黑名单：`rm -rf /`、`sudo`、`shutdown`、`reboot`、`> /dev/`

> **安全提示**：`shell=True` 存在命令注入风险。在教学代码中为了简化命令拼接而使用。在生产环境中，推荐使用参数数组方式调用命令，并对可执行命令建立白名单。

#### 3.3.4 保留工作树

```python
def keep(self, name: str) -> str:
    # 在索引中将 status 设为 "kept"
    item["status"] = "kept"
    item["kept_at"] = time.time()
    # 发出 worktree.keep 事件
    self.events.emit("worktree.keep", ...)
```

`keep` 的语义：工作树保留在磁盘上，标记为 "kept" 状态，供后续审查。

#### 3.3.5 移除工作树

```python
def remove(self, name: str, force: bool = False, complete_task: bool = False) -> str:
    # 发出 before 事件
    self.events.emit("worktree.remove.before", ...)

    # 执行 git worktree remove（force 时添加 --force 参数）
    args = ["worktree", "remove"]
    if force:
        args.append("--force")
    args.append(wt["path"])
    self._run_git(args)

    # 可选：完成绑定的任务
    if complete_task and wt.get("task_id") is not None:
        self.tasks.update(task_id, status="completed")
        self.tasks.unbind_worktree(task_id)
        self.events.emit("task.completed", ...)

    # 在索引中将状态设为 "removed"（不从数组中删除，而是标记）
    item["status"] = "removed"
    item["removed_at"] = time.time()

    # 发出 after 事件
    self.events.emit("worktree.remove.after", ...)
```

关键实现细节：
- `force=True` 时传递 `--force` 参数给 git，允许移除有未提交更改的工作树
- `complete_task=True` 时，同时将绑定的任务状态更新为 "completed"，并解除绑定
- 移除后索引中的条目**不会被删除**，而是将 status 标记为 "removed" 并记录 removed_at 时间戳
- 失败时发出 `worktree.remove.failed` 事件

#### 3.3.6 Git 命令封装

```python
def _run_git(self, args: list[str]) -> str:
    if not self.git_available:
        raise RuntimeError("Not in a git repository. worktree tools require git.")
    r = subprocess.run(
        ["git", *args],
        cwd=self.repo_root,       # 在仓库根目录执行
        capture_output=True,
        text=True,
        timeout=120,
    )
    if r.returncode != 0:
        msg = (r.stdout + r.stderr).strip()
        raise RuntimeError(msg or f"git {' '.join(args)} failed")
    return (r.stdout + r.stderr).strip() or "(no output)"
```

所有 git 命令都在 `self.repo_root`（仓库根目录）下执行，超时 120 秒。初始化时通过 `_is_git_repo()` 检测是否在 Git 仓库中，如果不是则所有 worktree 操作会抛出异常。

---

## 4. 工具集成

### 4.1 完整工具清单

源码注册了以下工具（除基础文件操作工具外）：

**任务管理工具**：

| 工具名 | 功能 | 必填参数 |
|--------|------|---------|
| `task_create` | 创建新任务 | subject |
| `task_list` | 列出所有任务 | （无） |
| `task_get` | 获取任务详情 | task_id |
| `task_update` | 更新任务状态或 owner | task_id |
| `task_bind_worktree` | 手动绑定任务到工作树 | task_id, worktree |

**工作树管理工具**：

| 工具名 | 功能 | 必填参数 |
|--------|------|---------|
| `worktree_create` | 创建工作树，可选绑定任务 | name |
| `worktree_list` | 列出索引中的所有工作树 | （无） |
| `worktree_status` | 查看某个工作树的 git status | name |
| `worktree_run` | 在工作树中执行命令 | name, command |
| `worktree_remove` | 移除工作树 | name |
| `worktree_keep` | 标记工作树为保留 | name |
| `worktree_events` | 查看最近的生命周期事件 | （可选 limit） |

### 4.2 工具处理器注册

```python
TOOL_HANDLERS = {
    # 基础工具
    "bash": lambda **kw: run_bash(kw["command"]),
    "read_file": lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file": lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    # 任务工具
    "task_create": lambda **kw: TASKS.create(kw["subject"], kw.get("description", "")),
    "task_list": lambda **kw: TASKS.list_all(),
    "task_get": lambda **kw: TASKS.get(kw["task_id"]),
    "task_update": lambda **kw: TASKS.update(kw["task_id"], kw.get("status"), kw.get("owner")),
    "task_bind_worktree": lambda **kw: TASKS.bind_worktree(kw["task_id"], kw["worktree"], kw.get("owner", "")),
    # 工作树工具
    "worktree_create": lambda **kw: WORKTREES.create(kw["name"], kw.get("task_id"), kw.get("base_ref", "HEAD")),
    "worktree_list": lambda **kw: WORKTREES.list_all(),
    "worktree_status": lambda **kw: WORKTREES.status(kw["name"]),
    "worktree_run": lambda **kw: WORKTREES.run(kw["name"], kw["command"]),
    "worktree_keep": lambda **kw: WORKTREES.keep(kw["name"]),
    "worktree_remove": lambda **kw: WORKTREES.remove(kw["name"], kw.get("force", False), kw.get("complete_task", False)),
    "worktree_events": lambda **kw: EVENTS.list_recent(kw.get("limit", 20)),
}
```

注意 `worktree_create` 支持三个参数：`name`（必填）、`task_id`（可选）、`base_ref`（可选，默认 "HEAD"）。

---

## 5. 完整示例

### 5.1 创建工作流

```
Lead:
  # 创建任务
  task_create(subject="重构认证模块")
  -> task #1（status: pending）

  # 创建工作树并绑定任务
  worktree_create(name="auth-refactor", task_id=1)
  -> 工作树 'auth-refactor' 已创建在 .worktrees/auth-refactor
  -> 任务 #1 绑定到工作树，状态自动变为 in_progress

[Alice 在工作树中工作]
  # 查看工作树状态
  worktree_status(name="auth-refactor")
  -> On branch wt/auth-refactor
  -> Clean worktree

  # 在工作树中执行命令
  worktree_run(name="auth-refactor", command="vim auth.py")
  -> [编辑文件]

  worktree_run(name="auth-refactor", command="git add -A && git commit -m 'refactor: 重新设计认证模块'")
  -> [wt/auth-refactor 1234abc] refactor: 重新设计认证模块

  # 任务完成，移除工作树并同时完成任务
  worktree_remove(name="auth-refactor", complete_task=true)
  -> Removed worktree 'auth-refactor'
  -> 任务 #1 状态更新为 completed，worktree 字段清空
```

### 5.2 keep 的工作流

```
  # 任务完成但保留工作树供审查
  worktree_keep(name="auth-refactor")
  -> 索引中 status 变为 "kept"，记录 kept_at 时间戳
  -> 发出 worktree.keep 事件

  # 后续手动移除
  worktree_remove(name="auth-refactor", force=true)
  -> Removed worktree 'auth-refactor'
```

### 5.3 事件流示例

执行上述流程后，`.worktrees/events.jsonl` 中记录：

```jsonl
{"event": "worktree.create.before", "ts": 1730000000.0, "task": {"id": 1}, "worktree": {"name": "auth-refactor", "base_ref": "HEAD"}}
{"event": "worktree.create.after", "ts": 1730000001.0, "task": {"id": 1}, "worktree": {"name": "auth-refactor", "path": "...", "branch": "wt/auth-refactor", "status": "active"}}
{"event": "worktree.remove.before", "ts": 1730003600.0, "task": {"id": 1}, "worktree": {"name": "auth-refactor", "path": "..."}}
{"event": "task.completed", "ts": 1730003601.0, "task": {"id": 1, "subject": "重构认证模块", "status": "completed"}, "worktree": {"name": "auth-refactor"}}
{"event": "worktree.remove.after", "ts": 1730003602.0, "task": {"id": 1}, "worktree": {"name": "auth-refactor", "path": "...", "status": "removed"}}
```

使用 `worktree_events` 工具可以随时查看这些事件。

---

## 6. 与 s11 的对比

| 方面 | s11 自主认领 | s12 工作树隔离 |
|------|------------|-------------|
| 执行环境 | 共享目录 | 独立 Git Worktree |
| 任务绑定 | owner 字段 | worktree + owner 双重绑定 |
| 冲突处理 | 无隔离机制 | 目录级别完全隔离 |
| 回滚能力 | 困难 | Git 分支级别回滚 |
| 状态追踪 | 任务文件 | 任务文件 + 事件流 |
| 并行安全 | 不安全 | 安全（独立文件系统） |
| 任务状态驱动 | 手动更新 | 绑定自动转 in_progress |

---

## 7. 常见问题

### Q1：不在 Git 仓库中会怎样？

`WorktreeManager.__init__` 中调用 `_is_git_repo()` 检测。不在 Git 仓库时，`self.git_available` 为 `False`，所有 worktree 操作抛出 `RuntimeError`。任务管理功能不受影响。

**降级方案**：非 Git 环境可用"子目录方案"——在主工作目录下创建独立子目录，手动复制所需文件。缺少 Git 的分支管理和硬链接效率，但文件级别仍能隔离：

```python
def create_isolated_dir(name: str) -> str:
    """降级方案：在非 Git 环境下创建隔离目录"""
    isolated_dir = Path(".workspaces") / name
    isolated_dir.mkdir(parents=True, exist_ok=True)
    import shutil
    for f in Path(".").glob("*.py"):
        shutil.copy2(f, isolated_dir / f.name)
    return str(isolated_dir)
```

实际项目中建议始终在 Git 仓库中工作。

### Q2：工作树有未提交的更改能移除吗？

默认不能。需传入 `force=True`，这会在 git 命令中添加 `--force` 参数。

### Q3：remove 和 keep 有什么区别？

- **remove** —— 执行 `git worktree remove` 删除磁盘目录，索引标记 status 为 "removed"
- **keep** —— 不删除目录，索引标记 status 为 "kept"，保留工作成果供后续审查

### Q4：如何查看某个工作树的 Git 状态？

使用 `worktree_status` 工具，在工作树目录下执行 `git status --short --branch`（详见 3.3.6 节 `_run_git` 封装）。

### Q5：`shell=True` 会有安全风险吗？

有风险。源码有两层防护：危险命令黑名单（`rm -rf /`、`sudo`、`shutdown`、`reboot`、`> /dev/`）和 300 秒超时。但黑名单无法覆盖所有危险命令。生产环境中推荐：使用参数数组调用、建立命令白名单、记录执行事件便于审计。

---

## 实战应用：为并行开发构建隔离环境

### 场景一：微服务并行开发

```
Lead:
  task_create(subject="用户服务") / task_create(subject="订单服务") / task_create(subject="支付服务")
  worktree_create(name="user-svc", task_id=1)
  worktree_create(name="order-svc", task_id=2)
  worktree_create(name="payment-svc", task_id=3)

Alice: worktree_run(name="user-svc", command="vim services/user.py") → commit
Bob:   worktree_run(name="order-svc", command="vim services/order.py") → commit
# 互不干扰

完成后：
  worktree_keep(name="user-svc")     # 保留供 code review
  worktree_remove(name="order-svc", complete_task=true)
```

### 场景二：A/B 测试隔离

```
worktree_create(name="sort-quick", task_id=1)
worktree_create(name="sort-merge", task_id=2)

worktree_run(name="sort-quick", command="python -c 'from sort import quicksort; ...'")
worktree_run(name="sort-merge", command="python -c 'from sort import mergesort; ...'")

worktree_keep(name="sort-quick")                      # 胜出方案保留
worktree_remove(name="sort-merge", force=true)         # 落选方案清理
```

### 场景三：CI 并行测试执行

```
worktree_create(name="test-unit") / worktree_create(name="test-integration") / worktree_create(name="test-e2e")

worktree_run(name="test-unit", command="pytest tests/unit/ -v")
worktree_run(name="test-integration", command="pytest tests/integration/ -v")
worktree_run(name="test-e2e", command="pytest tests/e2e/ -v")

# 收集结果后清理
worktree_remove(name="test-unit", force=true)
worktree_remove(name="test-integration", force=true)
worktree_remove(name="test-e2e", force=true)
```

### "从共享目录到工作树"迁移指南

| 当前状态 | 何时升级 | 迁移步骤 |
|----------|----------|----------|
| 1 个队友，串行任务 | 不需要 | 继续使用共享目录 |
| 2-3 个队友，修改不同文件 | 可暂缓 | 约定文件所有权即可 |
| 2+ 队友，可能修改同一文件 | **应该升级** | 为每个并行任务创建独立 worktree |
| A/B 对比实验 | **应该升级** | 两个 worktree 基于同一起点 |
| CI 并行测试 | **建议升级** | 每个测试分片一个 worktree |
| 需要完整审计链 | **必须升级** | EventBus 事件流提供完整可观测性 |

迁移清单：
1. 确认项目在 Git 仓库中（`git rev-parse --show-toplevel` 成功）
2. `.tasks/` 路径从 `WORKDIR` 改为 `REPO_ROOT`
3. `claim_task` 成功后自动调用 `worktree_create` 绑定
4. 队友的 `bash` 工具替换为 `worktree_run`
5. 任务完成时调用 `worktree_remove(complete_task=true)` 清理

---

## 8. 小结

### 8.1 核心要点

| 要点 | 说明 |
|------|------|
| **双面架构** | 任务是控制面，工作树是执行面，通过 task_id 绑定 |
| **Git Worktree** | 用 Git 原生能力实现目录级隔离 |
| **状态机** | active -> kept / removed，pending -> in_progress（自动） |
| **事件流** | append-only 日志记录完整生命周期，提供可观测性 |
| **索引持久化** | `.worktrees/index.json` 记录所有工作树元数据 |
| **安全防护** | 名称校验、危险命令拦截、超时保护 |

### 8.2 工作流模板

```
# 1. 创建任务
task_create(subject="任务描述")

# 2. 创建工作树并绑定（自动将任务变为 in_progress）
worktree_create(name="task-name", task_id=1)

# 3. 在工作树中工作
worktree_run(name="task-name", command="...")

# 4a. 完成：移除工作树 + 完成任务
worktree_remove(name="task-name", complete_task=true)

# 4b. 或保留工作树供审查
worktree_keep(name="task-name")
```

### 8.3 关键洞察

```
工作树隔离的核心价值：
1. 完全隔离 —— 独立的文件副本，互不干扰
2. 安全操作 —— Git 原生支持，稳定可靠
3. 易于回滚 —— 分支级别的版本控制
4. 可观测 —— 完整的事件流追踪

设计原则：
- 任务管理目标（控制面），工作树管理执行（执行面），通过 task ID 绑定
- 事件记录完整生命周期
```

---

## 9. 完整课程总结

恭喜！你已经完成了全部 12 章的学习。

### 知识体系回顾

```
第一阶段：核心循环（s01-s02）
├── s01: Agent 循环 ── 最小可用 Agent
└── s02: 工具系统 ── 多工具分发器

第二阶段：规划与知识（s03-s06）
├── s03: TodoWrite ── 任务规划
├── s04: Subagents ── 上下文隔离
├── s05: Skills ── 按需加载知识
└── s06: Context Compact ── 无限会话

第三阶段：持久化（s07-s08）
├── s07: Tasks ── 任务图持久化
└── s08: Background Tasks ── 后台并发

第四阶段：多 Agent 协作（s09-s12）
├── s09: Agent Teams ── 持久化队友
├── s10: Team Protocols ── 协作协议
├── s11: Autonomous Agents ── 自主认领
└── s12: Worktree Isolation ── 执行隔离
```

### 你现在已经能够：

1. 从零构建一个完整的 AI 编程 Agent
2. 设计多层次的工具系统
3. 实现上下文压缩和知识管理
4. 构建持久化的任务系统
5. 创建多 Agent 协作团队
6. 实现完全隔离的执行环境

### 下一步建议

1. **阅读整合源码** —— 运行 `agents/s_full.py`，观察 12 章全部能力的整合运行效果
2. **扩展工具系统** —— 添加自定义工具（如 `web_search`、`database_query`），实践 s02 的工具注册和处理器模式
3. **研究 MCP 协议** —— 了解 Model Context Protocol 如何让 Agent 动态发现外部工具，与硬编码工具列表对比
4. **优化上下文策略** —— 结合 s05 技能系统和 s06 压缩策略，设计完整的知识与上下文管理方案
5. **实践多 Agent 协作** —— 使用 s09-s12 的知识，构建自动分配、隔离执行、合并结果的多 Agent 团队
6. **分享改进** —— 将改进贡献给社区，或在个人项目中应用

---

## 10. 练习

### 练习 1：工作树状态看板

创建一个 `worktree_dashboard` 工具，汇总展示所有工作树的状态、绑定任务、最近事件。

### 练习 2：工作树合并

实现工作树合并功能：将完成的工作树分支合并回主分支，然后自动移除工作树。

### 练习 3：事件查询增强

给 EventBus 添加按事件类型、时间范围过滤的能力，支持 `worktree_events(event_type="worktree.create.after", since=1730000000)` 这样的查询。

### 练习 4：并行执行

实现 `run_parallel` 方法，在多个工作树中同时执行命令并收集结果（提示：使用 `subprocess.Popen` 或 `concurrent.futures`）。

---

< 上一章：[s11-自主 Agent](./s11-autonomous-agents.md) | [目录](./index.md) | 完成 >
