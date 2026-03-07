# s12：工作树隔离 —— 完美的终点

> **核心口号**：*"各自工作在自己的目录，互不干扰"*

> **学习目标**：理解执行隔离的必要性，掌握 Git Worktree 机制，学会完整的事件流

---

## 学习目标

完成本章后，你将能够：

1. **理解隔离的价值** —— 为什么需要独立的执行环境
2. **掌握 Git Worktree** —— 创建隔离的工作目录
3. **实现任务-工作树绑定** —— 任务与目录的关联
4. **设计事件流** —— 完整的生命周期追踪

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

- 能解释“共享目录冲突”与“工作树隔离”在协作稳定性上的差异。
- 能独立完成“创建工作树 → 绑定任务 → 执行命令 → 移除回收”流程。
- 能说清事件流在审计与故障恢复中的价值。

---

## 1. 问题的本质

### 1.1 共享目录的问题

在 s11 中，多个队友自主认领任务：

```
工作目录: /workspace/

Alice (认领任务 #1: 修改 auth.py):
  edit_file("auth.py", ...)
  write_file("auth.py", modified_content)

Bob (认领任务 #2: 同时修改 auth.py):
  edit_file("auth.py", ...)
  write_file("auth.py", other_modified_content)

结果: Bob 的修改覆盖了 Alice 的修改！
```

**问题**：
1. **文件冲突**：多人同时修改同一文件
2. **状态污染**：未提交的更改混在一起
3. **难以回滚**：无法单独撤销某个任务
4. **测试干扰**：一个任务的测试影响另一个

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

---

## 2. Git Worktree 基础

### 2.1 什么是 Worktree

Git Worktree 允许同一仓库的多个检出版本：

```bash
# 创建新的 worktree
git worktree add .worktrees/auth-refactor -b wt/auth-refactor HEAD

# 目录结构：
# main/           ← 主工作树
# .worktrees/auth-refactor/  ← 新工作树
#   └── .git 文件指向 main/.git/worktrees/auth-refactor/

# 列出所有 worktree
git worktree list

# 删除 worktree
git worktree remove .worktrees/auth-refactor
```

### 2.2 Worktree 的优势

| 特性 | 说明 |
|------|------|
| **隔离** | 完全独立的文件副本 |
| **共享历史** | 共享 .git 目录，节省空间 |
| **独立分支** | 每个 worktree 有自己的分支 |
| **原子操作** | Git 原生支持，安全可靠 |

---

## 3. 工作树管理器

### 3.1 核心类

```python
import json
import subprocess
import time
from pathlib import Path
from typing import Dict, Optional

class WorktreeManager:
    def __init__(self, worktrees_dir: Path, tasks_manager):
        self.dir = worktrees_dir
        self.dir.mkdir(exist_ok=True)
        self.tasks = tasks_manager

        # 索引文件
        self.index_path = self.dir / "index.json"
        self.index = self._load_index()

        # 事件流
        self.events_path = self.dir / "events.jsonl"

    def _load_index(self) -> Dict:
        """加载工作树索引"""
        if self.index_path.exists():
            return json.loads(self.index_path.read_text(encoding="utf-8"))
        return {"worktrees": {}}

    def _save_index(self):
        """保存索引"""
        self.index_path.write_text(
            json.dumps(self.index, indent=2, ensure_ascii=False),
            encoding="utf-8"
        )

    def _emit_event(self, event_type: str, data: Dict):
        """写入事件流"""
        event = {
            "event": event_type,
            "timestamp": int(time.time()),
            **data
        }
        with open(self.events_path, "a", encoding="utf-8") as f:
            f.write(json.dumps(event, ensure_ascii=False) + "\n")

    def _run_git(self, args: list) -> str:
        """运行 Git 命令"""
        result = subprocess.run(
            ["git"] + args,
            capture_output=True,
            text=True,
            check=True
        )
        return result.stdout.strip()
```

### 3.2 创建工作树

```python
def create(
    self,
    name: str,
    task_id: Optional[int] = None,
    branch: Optional[str] = None
) -> str:
    """创建新工作树"""
    if name in self.index["worktrees"]:
        return f"工作树 '{name}' 已存在"

    # 创建分支名
    if branch is None:
        branch = f"wt/{name}"

    # 创建工作树路径
    worktree_path = self.dir / name

    try:
        # 使用 git worktree add
        self._run_git([
            "worktree", "add",
            "-b", branch,
            str(worktree_path),
            "HEAD"
        ])

        # 更新索引
        self.index["worktrees"][name] = {
            "path": str(worktree_path),
            "branch": branch,
            "status": "active",
            "created_at": int(time.time())
        }
        self._save_index()

        # 绑定任务
        if task_id is not None:
            self._bind_task(task_id, name)

        # 发出事件
        self._emit_event("worktree.create.after", {
            "name": name,
            "path": str(worktree_path),
            "branch": branch
        })

        return f"工作树 '{name}' 已创建在 {worktree_path}"

    except subprocess.CalledProcessError as e:
        return f"创建工作树失败: {e.stderr}"
```

### 3.3 任务绑定

```python
def _bind_task(self, task_id: int, worktree_name: str):
    """绑定任务到工作树"""
    task = self.tasks._load(task_id)

    # 设置工作树
    task["worktree"] = worktree_name

    # 如果任务是 pending，自动转为 in_progress
    if task["status"] == "pending":
        task["status"] = "in_progress"

    # 保存任务
    self.tasks._save(task)

    # 更新工作树索引
    self.index["worktrees"][worktree_name]["task_id"] = task_id
    self._save_index()

    # 发出事件
    self._emit_event("task.bound", {
        "task_id": task_id,
        "worktree": worktree_name
    })

    return f"任务 #{task_id} 绑定到工作树 '{worktree_name}'"
```

### 3.4 在工作树中执行命令

```python
def run_in_worktree(self, name: str, command: str) -> str:
    """在工作树中执行命令"""
    if name not in self.index["worktrees"]:
        return f"工作树不存在: {name}"

    worktree = self.index["worktrees"][name]
    worktree_path = worktree["path"]

    if worktree["status"] != "active":
        return f"工作树 '{name}' 状态为 {worktree['status']}，无法执行"

    try:
        result = subprocess.run(
            command,
            shell=True,
            cwd=worktree_path,  # 关键：在工作树目录中执行
            capture_output=True,
            text=True,
            timeout=300
        )
        output = (result.stdout + result.stderr).strip()[:50000]
        return output
    except subprocess.TimeoutExpired:
        return f"命令超时"
    except Exception as e:
        return f"执行失败: {str(e)}"
```

**准确性补充**：
- 示例里使用 `shell=True` 是为了教学简化命令拼接。
- 在生产环境中，优先使用参数数组方式调用命令，并对命令白名单做限制，避免命令注入风险。

### 3.5 保留和移除

```python
def keep(self, name: str) -> str:
    """保留工作树（标记为 kept）"""
    if name not in self.index["worktrees"]:
        return f"工作树不存在: {name}"

    self.index["worktrees"][name]["status"] = "kept"
    self._save_index()

    self._emit_event("worktree.keep", {
        "name": name
    })

    return f"工作树 '{name}' 已标记为保留"

def remove(
    self,
    name: str,
    force: bool = False,
    complete_task: bool = False
) -> str:
    """移除工作树"""
    if name not in self.index["worktrees"]:
        return f"工作树不存在: {name}"

    worktree = self.index["worktrees"][name]
    worktree_path = worktree["path"]

    # 发出事件
    self._emit_event("worktree.remove.before", {
        "name": name,
        "path": worktree_path
    })

    try:
        # 移除 Git worktree
        self._run_git(["worktree", "remove", str(worktree_path)])

        # 完成绑定的任务
        if complete_task and worktree.get("task_id"):
            task_id = worktree["task_id"]

            # 更新任务状态
            self.tasks.update(task_id, status="completed")

            # 解除绑定
            task = self.tasks._load(task_id)
            task["worktree"] = ""
            self.tasks._save(task)

            # 发出事件
            self._emit_event("task.completed", {
                "task_id": task_id,
                "worktree": name
            })

        # 从索引中移除
        del self.index["worktrees"][name]
        self._save_index()

        # 发出事件
        self._emit_event("worktree.remove.after", {
            "name": name,
            "path": worktree_path
        })

        return f"工作树 '{name}' 已移除"

    except subprocess.CalledProcessError as e:
        return f"移除工作树失败: {e.stderr}"
```

---

## 4. 工具集成

### 4.1 工具定义

```python
TOOLS.extend([
    {
        "name": "worktree_create",
        "description": "创建新的工作树",
        "input_schema": {
            "type": "object",
            "properties": {
                "name": {"type": "string"},
                "task_id": {"type": "integer"},
                "branch": {"type": "string"}
            },
            "required": ["name"]
        }
    },
    {
        "name": "worktree_run",
        "description": "在工作树中执行命令",
        "input_schema": {
            "type": "object",
            "properties": {
                "worktree": {"type": "string"},
                "command": {"type": "string"}
            },
            "required": ["worktree", "command"]
        }
    },
    {
        "name": "worktree_keep",
        "description": "保留工作树",
        "input_schema": {
            "type": "object",
            "properties": {
                "name": {"type": "string"}
            },
            "required": ["name"]
        }
    },
    {
        "name": "worktree_remove",
        "description": "移除工作树",
        "input_schema": {
            "type": "object",
            "properties": {
                "name": {"type": "string"},
                "force": {"type": "boolean"},
                "complete_task": {"type": "boolean"}
            },
            "required": ["name"]
        }
    },
    {
        "name": "worktree_list",
        "description": "列出所有工作树",
        "input_schema": {
            "type": "object",
            "properties": {},
            "required": []
        }
    },
])
```

### 4.2 注册处理器

```python
WORKTREES = WorktreeManager(Path(".worktrees"), TASKS)

TOOL_HANDLERS.update({
    "worktree_create": lambda **kw: WORKTREES.create(
        kw["name"],
        kw.get("task_id"),
        kw.get("branch")
    ),
    "worktree_run": lambda **kw: WORKTREES.run_in_worktree(
        kw["worktree"],
        kw["command"]
    ),
    "worktree_keep": lambda **kw: WORKTREES.keep(kw["name"]),
    "worktree_remove": lambda **kw: WORKTREES.remove(
        kw["name"],
        kw.get("force", False),
        kw.get("complete_task", False)
    ),
    "worktree_list": lambda **kw: json.dumps(WORKTREES.index, indent=2),
})
```

---

## 5. 完整示例

### 5.1 创建工作流

```
Lead:
  # 创建任务
  task_create(subject="重构认证模块")
  → task #1

  # 创建工作树并绑定任务
  worktree_create(name="auth-refactor", task_id=1)
  → 工作树 'auth-refactor' 已创建
  → 任务 #1 绑定到工作树，状态变为 in_progress

[Alice 认领任务后]
  # 在工作树中工作
  worktree_run(worktree="auth-refactor", command="git status")
  → On branch wt/auth-refactor
  → Changes not staged...

  worktree_run(worktree="auth-refactor", command="vim auth.py")
  → [编辑文件]

  worktree_run(worktree="auth-refactor", command="git add -A && git commit -m 'refactor: 重新设计认证模块'")
  → [wt/auth-refactor 1234abc] refactor: 重新设计认证模块

  # 任务完成
  worktree_remove(name="auth-refactor", complete_task=true)
  → 工作树 'auth-refactor' 已移除
  → 任务 #1 状态更新为 completed
```

### 5.2 事件流

```jsonl
{"event": "worktree.create.after", "name": "auth-refactor", "timestamp": 1730000000}
{"event": "task.bound", "task_id": 1, "worktree": "auth-refactor", "timestamp": 1730000001}
{"event": "task.completed", "task_id": 1, "worktree": "auth-refactor", "timestamp": 1730003600}
{"event": "worktree.remove.before", "name": "auth-refactor", "timestamp": 1730003601}
{"event": "worktree.remove.after", "name": "auth-refactor", "timestamp": 1730003602}
```

---

## 6. 与 s11 的对比

| 方面 | s11 自主 | s12 隔离 |
|------|----------|---------|
| 执行环境 | 共享目录 | 独立工作树 |
| 任务绑定 | owner 字段 | worktree + owner |
| 冲突处理 | 无隔离 | 完全隔离 |
| 回滚 | 困难 | Git 原生支持 |
| 状态追踪 | 任务文件 | + 事件流 |

---

## 7. 常见问题

### Q1：如何处理工作树中的未提交更改？

```python
def remove(self, name: str, force: bool = False, ...):
    worktree = self.index["worktrees"][name]
    worktree_path = worktree["path"]

    # 检查是否有未提交的更改
    if not force:
        result = subprocess.run(
            ["git", "status", "--porcelain"],
            cwd=worktree_path,
            capture_output=True,
            text=True
        )
        if result.stdout.strip():
            return f"工作树有未提交的更改，使用 force=true 强制删除"

    # 移除工作树
    ...
```

### Q2：如何恢复崩溃后的状态？

```python
def recover(self) -> str:
    """从索引恢复状态"""
    recovered = []

    for name, worktree in self.index["worktrees"].items():
        if worktree["status"] == "active":
            # 检查工作树是否仍然存在
            if not Path(worktree["path"]).exists():
                # 工作树丢失，标记为失效
                worktree["status"] = "lost"
                recovered.append(f"{name}: 标记为丢失")
            else:
                recovered.append(f"{name}: 正常")

    self._save_index()
    return "\n".join(recovered)
```

### Q3：多个队友可以使用同一工作树吗？

**答案**：不推荐。每个任务应该有自己的工作树。如果需要协作，使用分支而不是工作树。

### Q4：如何限制工作树数量？

```python
MAX_ACTIVE_WORKTREES = 10

def create(self, name: str, ...) -> str:
    # 检查活动工作树数量
    active = sum(
        1 for wt in self.index["worktrees"].values()
        if wt["status"] == "active"
    )

    if active >= MAX_ACTIVE_WORKTREES:
        return f"活动工作树已达上限 ({MAX_ACTIVE_WORKTREES})"

    # 继续创建...
```

### Q5：`shell=True` 会有安全风险吗？

**答案**：有风险，尤其当命令包含用户输入时。推荐做法：

1. 优先使用参数数组，不拼接原始字符串。
2. 对可执行命令建立白名单。
3. 为执行目录和超时时间设置硬限制。
4. 记录命令执行事件，便于审计与追踪。

---

## 8. 小结

### 8.1 核心要点

| 要点 | 说明 |
|------|------|
| **Git Worktree** | 隔离的执行环境 |
| **任务绑定** | 任务与工作树关联 |
| **状态管理** | active/kept/removed |
| **事件流** | 完整的生命周期追踪 |

### 8.2 工作流模板

```python
# 1. 创建任务
task_create(subject="任务描述")

# 2. 创建工作树并绑定
worktree_create(name="task-name", task_id=1)

# 3. 在工作树中工作
worktree_run(worktree="task-name", command="...")

# 4. 完成
worktree_remove(name="task-name", complete_task=true)
```

### 8.3 关键洞察

```
工作树隔离的核心价值：
1. 完全隔离：独立的文件副本
2. 安全操作：Git 原生支持
3. 易于回滚：分支级别的版本控制
4. 可追踪：完整的事件流

设计原则：
- 任务管理目标
- 工作树管理执行
- 通过 ID 绑定
- 事件记录生命周期
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

1. ✅ 从零构建一个完整的 AI 编程 Agent
2. ✅ 设计多层次的工具系统
3. ✅ 实现上下文压缩和知识管理
4. ✅ 构建持久化的任务系统
5. ✅ 创建多 Agent 协作团队
6. ✅ 实现完全隔离的执行环境

### 下一步建议

1. **实践**：运行 `agents/s_full.py`，观察完整系统
2. **扩展**：添加新的工具和技能
3. **优化**：根据实际需求调整参数和策略
4. **分享**：将你的改进贡献给社区

---

## 10. 练习

### 练习 1：工作树模板

创建工作树模板，新工作树自动包含特定文件。

### 练习 2：工作树合并

实现工作树合并功能，将完成的工作树合并回主分支。

### 练习 3：资源限制

实现工作树的资源限制（磁盘空间、文件数量等）。

### 练习 4：事件查询

添加事件查询工具，支持按类型、时间范围过滤事件。

---

< 上一章：[s11-自主 Agent](./s11-autonomous-agents.md) | [目录](./index.md) | 完成 >
