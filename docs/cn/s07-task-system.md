# s07：任务系统 —— 持久化的任务图

> **核心口号**：*"大目标拆成小任务，排序，持久化"*

> **学习目标**：理解任务图的设计，掌握 DAG 依赖关系，学会跨会话的状态管理

---

## 学习目标

完成本章后，你将能够：

1. **理解任务与 Todo 的区别** —— 为什么需要持久化
2. **掌握 DAG 设计** —— 有向无环图的依赖关系
3. **实现 TaskManager** —— 完整的 CRUD 操作
4. **理解状态传播** —— 完成任务如何解除依赖

---

## 0. 上手演练（建议先做）

先运行任务系统示例，再阅读 DAG 与状态传播：

```bash
uv run python agents/s07_task_system.py
```

建议观察：

1. 任务文件如何在磁盘上持久化并跨轮次读取。
2. `blockedBy`/`blocks` 如何表达依赖关系。
3. 一个任务状态变化后，下游任务可执行性如何更新。

### 本章完成标准

- 能用真实任务举例说明 DAG 比扁平 Todo 更适合复杂流程。
- 能独立完成任务创建、依赖建立、状态推进与恢复。
- 能解释“持久化 + 依赖图”对长周期任务的价值。

---

## 1. 从 Todo 到 Task

### 1.1 回顾 s03 的 TodoManager

```python
# TodoManager (s03)
class TodoManager:
    def __init__(self):
        self.items: List[Dict] = []  # 只存在内存中
```

**问题**：
1. **只存在内存**：上下文压缩后丢失
2. **没有依赖关系**：无法表达"任务 B 等待任务 A"
3. **扁平结构**：无法表示复杂的任务层次

### 1.2 Task 的改进

```python
# TaskManager (s07)
.tasks/
├── task_1.json  # 每个任务独立文件
├── task_2.json
├── task_3.json
└── task_4.json

# 任务文件内容
{
  "id": 1,
  "subject": "实现用户认证",
  "description": "...",
  "status": "pending",
  "blockedBy": [],      # 等待哪些任务
  "blocks": [2, 3],     # 阻塞哪些任务
  "owner": "",
  "created_at": 1234567890
}
```

**改进**：
1. **持久化**：存储在磁盘，跨会话存活
2. **依赖关系**：`blockedBy` 和 `blocks` 字段
3. **可恢复**：压缩后可从磁盘恢复状态

---

## 2. 任务图（DAG）

### 2.1 什么是 DAG

DAG = Directed Acyclic Graph（有向无环图）

```
任务依赖图示例：

        task_1 (setup)
         /      \
        v        v
  task_2 (auth)  task_3 (db)
        \        /
         v      v
       task_4 (test)

属性：
- 有向：箭头表示依赖方向
- 无环：没有循环依赖
- 传递：如果 1→2 且 2→4，则 1→4
```

### 2.2 依赖关系表示

```python
# task_1.json
{
  "id": 1,
  "subject": "项目初始化",
  "status": "completed",
  "blockedBy": [],   # 不等待任何任务
  "blocks": [2, 3]   # 完成后解除 2 和 3 的阻塞
}

# task_2.json
{
  "id": 2,
  "subject": "实现认证模块",
  "status": "pending",
  "blockedBy": [1],  # 等待任务 1
  "blocks": [4]      # 完成后解除 4 的阻塞
}

# task_3.json
{
  "id": 3,
  "subject": "实现数据库模块",
  "status": "pending",
  "blockedBy": [1],  # 等待任务 1
  "blocks": [4]      # 完成后解除 4 的阻塞
}

# task_4.json
{
  "id": 4,
  "subject": "编写集成测试",
  "status": "pending",
  "blockedBy": [2, 3], # 等待任务 2 和 3
  "blocks": []
}
```

### 2.3 任务状态机

```
                    ┌─────────────┐
                    │   pending   │ ◄──── 创建
                    └──────┬──────┘
                           │ assign
                           ▼
                    ┌─────────────┐
                    │in_progress  │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │             │
                  完成           取消
                    │             │
                    ▼             ▼
              ┌──────────┐   ┌──────────┐
              │completed │   │ cancelled │
              └──────────┘   └──────────┘
```

---

## 3. TaskManager 实现

### 3.1 核心类

```python
import json
import time
from pathlib import Path
from typing import List, Dict, Optional, Literal

TaskStatus = Literal["pending", "in_progress", "completed", "cancelled"]

class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(exist_ok=True)
        self._next_id = self._max_id() + 1

    def _max_id(self) -> int:
        """获取当前最大的任务 ID"""
        max_id = 0
        for f in self.dir.glob("task_*.json"):
            task = json.loads(f.read_text())
            max_id = max(max_id, task["id"])
        return max_id

    def _task_path(self, task_id: int) -> Path:
        """获取任务文件路径"""
        return self.dir / f"task_{task_id}.json"

    def _load(self, task_id: int) -> Dict:
        """加载任务"""
        path = self._task_path(task_id)
        if not path.exists():
            raise ValueError(f"任务不存在: {task_id}")
        return json.loads(path.read_text(encoding="utf-8"))

    def _save(self, task: Dict):
        """保存任务"""
        path = self._task_path(task["id"])
        path.write_text(json.dumps(task, indent=2, ensure_ascii=False), encoding="utf-8")
```

### 3.2 创建任务

```python
def create(self, subject: str, description: str = "") -> str:
    """创建新任务"""
    task = {
        "id": self._next_id,
        "subject": subject,
        "description": description,
        "status": "pending",
        "blockedBy": [],
        "blocks": [],
        "owner": "",
        "created_at": int(time.time()),
        "updated_at": int(time.time())
    }
    self._save(task)
    self._next_id += 1
    return json.dumps(task, indent=2, ensure_ascii=False)
```

### 3.3 更新任务

```python
def update(
    self,
    task_id: int,
    status: Optional[TaskStatus] = None,
    add_blocked_by: Optional[int] = None,
    add_blocks: Optional[int] = None
) -> str:
    """更新任务"""
    task = self._load(task_id)

    if status:
        task["status"] = status
        task["updated_at"] = int(time.time())

        # 如果完成任务，解除依赖
        if status == "completed":
            self._clear_dependency(task_id)

    if add_blocked_by is not None:
        if add_blocked_by not in task["blockedBy"]:
            task["blockedBy"].append(add_blocked_by)

    if add_blocks is not None:
        if add_blocks not in task["blocks"]:
            task["blocks"].append(add_blocks)

    self._save(task)
    return json.dumps(task, indent=2, ensure_ascii=False)

def _clear_dependency(self, completed_id: int):
    """清除已完成任务的依赖"""
    # 从所有任务的 blockedBy 中移除已完成任务的 ID
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            f.write_text(json.dumps(task, indent=2, ensure_ascii=False), encoding="utf-8")
```

### 3.4 查询任务

```python
def get(self, task_id: int) -> str:
    """获取单个任务"""
    return json.dumps(self._load(task_id), indent=2, ensure_ascii=False)

def list_all(self) -> str:
    """列出所有任务"""
    tasks = []
    for f in sorted(self.dir.glob("task_*.json")):
        task = json.loads(f.read_text())
        tasks.append(task)

    # 格式化输出
    lines = ["任务列表：", ""]
    for task in tasks:
        status_emoji = {
            "pending": "[ ]",
            "in_progress": "[>]",
            "completed": "[x]",
            "cancelled": "[_]"
        }
        emoji = status_emoji.get(task["status"], "[?]")
        owner = f" @{task['owner']}" if task.get("owner") else ""
        blocked = f" (等待: {task['blockedBy']})" if task["blockedBy"] else ""

        lines.append(f"{emoji} #{task['id']}: {task['subject']}{owner}{blocked}")

    return "\n".join(lines)

def get_ready(self) -> List[Dict]:
    """获取可以执行的任务（pending 且无阻塞）"""
    ready = []
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if (task["status"] == "pending"
                and not task.get("blockedBy")
                and not task.get("owner")):
            ready.append(task)
    return sorted(ready, key=lambda t: t["id"])
```

---

## 4. 工具集成

### 4.1 工具定义

```python
TOOLS = [
    # ... 其他工具 ...
    {
        "name": "task_create",
        "description": "创建新任务",
        "input_schema": {
            "type": "object",
            "properties": {
                "subject": {"type": "string"},
                "description": {"type": "string"}
            },
            "required": ["subject"]
        }
    },
    {
        "name": "task_update",
        "description": "更新任务状态或添加依赖",
        "input_schema": {
            "type": "object",
            "properties": {
                "task_id": {"type": "integer"},
                "status": {
                    "type": "string",
                    "enum": ["pending", "in_progress", "completed", "cancelled"]
                },
                "add_blocked_by": {"type": "integer"},
                "add_blocks": {"type": "integer"}
            },
            "required": ["task_id"]
        }
    },
    {
        "name": "task_get",
        "description": "获取任务详情",
        "input_schema": {
            "type": "object",
            "properties": {
                "task_id": {"type": "integer"}
            },
            "required": ["task_id"]
        }
    },
    {
        "name": "task_list",
        "description": "列出所有任务",
        "input_schema": {
            "type": "object",
            "properties": {},
            "required": []
        }
    },
]
```

### 4.2 注册处理器

```python
TASKS = TaskManager(Path(".tasks"))

TOOL_HANDLERS = {
    # ... 其他工具 ...
    "task_create": lambda **kw: TASKS.create(kw["subject"], kw.get("description", "")),
    "task_update": lambda **kw: TASKS.update(
        kw["task_id"],
        status=kw.get("status"),
        add_blocked_by=kw.get("add_blocked_by"),
        add_blocks=kw.get("add_blocks")
    ),
    "task_get": lambda **kw: TASKS.get(kw["task_id"]),
    "task_list": lambda **kw: TASKS.list_all(),
}
```

---

## 5. 完整示例

### 5.1 创建任务图

```
用户：创建一个登录功能的任务计划

Agent：
  task_create(subject="设计数据库 schema")
  → #1 created

  task_create(subject="实现用户模型")
  → #2 created

  task_create(subject="实现认证 API")
  → #3 created

  task_create(subject="编写登录页面")
  → #4 created

  task_create(subject="编写测试")
  → #5 created

  # 设置依赖关系
  task_update(task_id=2, add_blocked_by=1)  # 用户模型等待 schema
  task_update(task_id=3, add_blocked_by=2)  # API 等待用户模型
  task_update(task_id=4, add_blocked_by=3)  # 页面等待 API
  task_update(task_id=5, add_blocked_by=[3, 4])  # 测试等待 API 和页面

  task_list()
```

### 5.2 任务列表输出

```
任务列表：

[ ] #1: 设计数据库 schema
[ ] #2: 实现用户模型 (等待: [1])
[ ] #3: 实现认证 API (等待: [2])
[ ] #4: 编写登录页面 (等待: [3])
[ ] #5: 编写测试 (等待: [3, 4])
```

### 5.3 执行过程

```
# 开始任务 1
task_update(task_id=1, status="in_progress")
# ... 完成 schema 设计 ...
task_update(task_id=1, status="completed")

# 任务 1 完成后，自动解除任务 2 的阻塞
# 现在可以开始任务 2
task_update(task_id=2, status="in_progress")
# ... 完成用户模型 ...
task_update(task_id=2, status="completed")

# 任务 2 完成后，解除任务 3 的阻塞
task_update(task_id=3, status="in_progress")
# ... 完成 API ...
task_update(task_id=3, status="completed")

# 任务 3 完成后，同时解除任务 4 和 5 的阻塞
# 任务 4 和 5 可以并行执行！
```

---

## 6. 与 s03 的对比

| 特性 | Todo (s03) | Task (s07) |
|------|------------|------------|
| 存储 | 内存列表 | 磁盘文件 |
| 持久化 | 无 | 有 |
| 依赖关系 | 无 | blockedBy + blocks |
| 状态 | pending/in_progress/done | + cancelled |
| 所有权 | 无 | owner 字段 |
| 时间戳 | 无 | created_at + updated_at |
| 恢复性 | 压缩后丢失 | 可从磁盘恢复 |

---

## 7. 常见问题

### Q1：为什么每个任务一个文件？

**答案**：
1. **原子性**：更新单个任务不影响其他任务
2. **并发**：多个 Agent 可以同时更新不同任务
3. **简单**：文件系统天然支持锁和事务

替代方案：单一 JSON 文件（更简单但不适合并发）

### Q2：如何防止循环依赖？

```python
def _check_cycle(self, task_id: int, blocked_by: int) -> bool:
    """检查是否会形成循环"""
    # DFS 检查是否存在路径从 blocked_by 到 task_id
    visited = set()

    def dfs(current):
        if current == task_id:
            return True  # 找到循环
        if current in visited:
            return False
        visited.add(current)

        task = self._load(current)
        for dep in task.get("blockedBy", []):
            if dfs(dep):
                return True
        return False

    return dfs(blocked_by)
```

### Q3：删除任务怎么处理？

```python
def delete(self, task_id: int) -> str:
    """删除任务（只能删除 pending 状态的任务）"""
    task = self._load(task_id)

    if task["status"] != "pending":
        raise ValueError(f"只能删除 pending 状态的任务，当前状态: {task['status']}")

    # 检查是否有其他任务依赖此任务
    for f in self.dir.glob("task_*.json"):
        t = json.loads(f.read_text())
        if task_id in t.get("blockedBy", []):
            raise ValueError(f"任务 #{t['id']} 依赖此任务，无法删除")

    # 删除文件
    self._task_path(task_id).unlink()
    return f"任务 #{task_id} 已删除"
```

---

## 8. 最佳实践

### 8.1 任务粒度

```
太粗：
- "实现登录功能"（太大，无法跟踪）

太细：
- "创建文件 auth.py"
- "定义 User 类"
- "实现 login 方法"
- "添加参数验证"
- ...（太多，管理复杂）

合适：
- "设计数据库 schema"
- "实现用户模型"
- "实现认证 API"
- "编写登录页面"
- "编写集成测试"
```

### 8.2 依赖设计原则

```
✅ 好的依赖：
- 逻辑依赖：B 真正需要 A 的结果
- 明确：依赖关系清晰可见
- 最小化：只依赖必需的任务

❌ 坏的依赖：
- 顺序偏好：A 应该在 B 之前（但不需要）
- 隐式依赖：没有显式声明
- 过度依赖：太多依赖关系
```

### 8.3 任务描述规范

```python
# ✅ 好的任务描述
{
    "subject": "实现用户认证 API",
    "description": """
    实现 /api/login 和 /api/logout 端点：
    - 使用 JWT 进行认证
    - 支持刷新 token
    - 返回标准的错误响应
    依赖：用户模型 (task #2)
    """
}

# ❌ 差的任务描述
{
    "subject": "API",
    "description": "做 API"
}
```

---

## 9. 小结

### 9.1 核心要点

| 要点 | 说明 |
|------|------|
| **持久化** | 任务存储在磁盘，跨会话存活 |
| **DAG** | 有向无环图表示依赖关系 |
| **状态传播** | 完成任务自动解除依赖 |
| **原子操作** | 每个任务独立文件 |

### 9.2 代码模板

```python
# 创建任务
task_create(subject="任务名称", description="详细描述")

# 设置依赖
task_update(task_id=2, add_blocked_by=1)

# 开始任务
task_update(task_id=1, status="in_progress")

# 完成任务（自动解除依赖）
task_update(task_id=1, status="completed")

# 列出任务
task_list()
```

### 9.3 关键洞察

```
任务系统的核心价值：
1. 将复杂的任务分解为可管理的单元
2. 明确任务之间的依赖关系
3. 支持任务的并行执行
4. 跨会话保持状态

设计原则：
- 每个任务一个文件（原子性）
- DAG 结构（无循环依赖）
- 完成自动解除依赖（状态传播）
- pending + 无阻塞 = 可执行
```

---

## 10. 练习

### 练习 1：任务优先级

添加 `priority` 字段（high/medium/low），在 `get_ready()` 中按优先级排序。

### 练习 2：任务标签

添加 `tags` 数组字段，支持按标签过滤任务。

### 练习 3：任务搜索

实现 `task_search` 工具，支持按关键词搜索任务。

### 练习 4：依赖可视化

生成任务依赖图的 Graphviz DOT 格式输出。

---

## 11. 下一步

现在任务系统可以持久化到磁盘了。但执行长时间任务（如运行测试）仍然会阻塞 Agent 的思考。

在 [s08：后台任务](./s08-background-tasks.md) 中，我们将学习：
- 如何在后台运行命令
- 通知队列机制
- 并发执行多个任务

---

< 上一章：[s06-上下文压缩](./s06-context-compact.md) | [目录](./index.md) | 下一章：[s08-后台任务](./s08-background-tasks.md) >
