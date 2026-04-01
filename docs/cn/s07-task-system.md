# s07：任务系统 —— 持久化的任务图

> **核心口号**：*"大目标拆成小任务，排序，持久化"*

> **关键洞察**：*"State that survives compression -- because it's outside the conversation."*

---

## 学习目标

完成本章后，你将能够：

1. **理解任务与 Todo 的区别** —— 为什么需要持久化到磁盘
2. **掌握 DAG 设计** —— 有向无环图的依赖关系如何用 JSON 文件表示
3. **实现 TaskManager** —— 完整的 CRUD 操作与依赖管理
4. **理解状态传播** —— 完成任务如何通过全局扫描自动解除下游阻塞

---

## 0. 上手演练（建议先做）

先运行任务系统示例，再阅读 DAG 与状态传播：

```bash
uv run python agents/s07_task_system.py
```

建议观察：

1. 在项目根目录下观察 `.tasks/` 目录如何自动创建，以及 `task_*.json` 文件的生成。
2. 在 Agent 中输入"创建一个项目初始化的任务"，观察 Agent 如何调用 `task_create` 工具。
3. 多次创建任务并建立依赖关系，用 `task_list` 查看 `[ ]`、`[>]`、`[x]` 状态标记。
4. 当一个被依赖的任务被标记为 `completed` 后，下游任务的 `blockedBy` 列表如何自动清空。

### 本章完成标准

- 能用真实任务举例说明 DAG 比扁平 Todo 更适合复杂流程。
- 能独立完成任务创建、依赖建立、状态推进与恢复。
- 能解释"持久化 + 依赖图"对长周期任务的价值。
- 能精确描述 `_clear_dependency` 的全局扫描逻辑。

---

## 1. 从 Todo 到 Task

### 1.1 回顾 s03 的 TodoManager

```python
# s03_todo_write.py 中的 TodoManager
class TodoManager:
    def __init__(self):
        self.items: List[Dict] = []  # 只存在内存中
```

**s03 的三个限制**：

| 问题 | 具体表现 | 后果 |
|------|----------|------|
| 只存在内存 | 上下文压缩后 `self.items` 丢失 | Agent 忘记待办事项 |
| 没有依赖关系 | 无法表达"任务 B 等待任务 A" | Agent 可能执行顺序错误 |
| 扁平结构 | 只有一个线性列表 | 无法表示复杂的任务层次和并行 |

### 1.2 Task 的改进

s07 的 `TaskManager` 针对以上每个问题做出了精确的改进：

```
.tasks/
├── task_1.json  # 每个任务独立文件
├── task_2.json
├── task_3.json
└── task_4.json
```

```json
// task_1.json 的实际结构（源码 s07_task_system.py 第 67-70 行）
{
  "id": 1,
  "subject": "实现用户认证",
  "description": "...",
  "status": "pending",
  "blockedBy": [],      // 等待哪些任务完成
  "blocks": [2, 3],     // 阻塞哪些任务（反向引用）
  "owner": ""           // 任务归属（为多 Agent 场景预留）
}
```

**三重改进**：

| 改进 | 机制 | 解决的问题 |
|------|------|-----------|
| 持久化 | 每个任务写入 `.tasks/task_N.json` | 上下文压缩后状态不丢失 |
| 依赖关系 | `blockedBy` + `blocks` 双向引用 | 表达任务间的前置/后续关系 |
| 可恢复 | Agent 重启时从磁盘读取所有 `task_*.json` | 跨会话保持状态 |

---

## 2. 任务图（DAG）

### 2.1 什么是 DAG

DAG = Directed Acyclic Graph（有向无环图）。

- **有向**：边有方向，表示"依赖"关系（A 完成后 B 才能开始）
- **无环**：不存在循环依赖（A 等 B、B 等 A 的情况不允许）
- **图**：节点是任务，边是依赖

```
任务依赖图示例（菱形结构）：

        task_1 (项目初始化)
         /            \
        v              v
  task_2 (认证模块)  task_3 (数据库模块)
        \              /
         v            v
        task_4 (集成测试)

含义：
  - task_2 和 task_3 都依赖 task_1（不能并行开始）
  - task_4 依赖 task_2 和 task_3（两个都完成才能开始）
  - task_2 和 task_3 之间无依赖（可以并行执行）
```

### 2.2 依赖关系的存储格式

DAG 在源码中通过 JSON 文件的 `blockedBy` 和 `blocks` 字段表示：

```json
// task_1.json — 根节点，不等待任何任务
{
  "id": 1, "subject": "项目初始化", "status": "completed",
  "blockedBy": [],
  "blocks": [2, 3]    // task_1 完成后会解除 task_2 和 task_3 的阻塞
}

// task_2.json — 等待 task_1
{
  "id": 2, "subject": "认证模块", "status": "pending",
  "blockedBy": [1],   // 等待任务 1
  "blocks": [4]       // 完成后解除 task_4 的阻塞
}

// task_3.json — 等待 task_1
{
  "id": 3, "subject": "数据库模块", "status": "pending",
  "blockedBy": [1],
  "blocks": [4]
}

// task_4.json — 等待 task_2 和 task_3
{
  "id": 4, "subject": "集成测试", "status": "pending",
  "blockedBy": [2, 3],
  "blocks": []
}
```

**双向引用的设计意图**：

- `blockedBy`：记录"我被谁阻塞"—— Agent 查询时可直接判断是否可执行
- `blocks`：记录"我阻塞了谁"—— 用于建立依赖时自动维护反向引用（见第 3.3 节 `add_blocks` 逻辑）

两个方向的数据都需要维护，但查询时只需看 `blockedBy` 即可判断可执行性。

### 2.3 任务状态机

源码中任务有三个合法状态（见 `s07_task_system.py` 第 82-83 行）：

```
                    ┌─────────────┐
                    │   pending   │ ◄──── task_create() 创建
                    └──────┬──────┘
                           │ task_update(status="in_progress")
                           ▼
                    ┌─────────────┐
                    │ in_progress │
                    └──────┬──────┘
                           │ task_update(status="completed")
                           ▼
                    ┌─────────────┐
                    │  completed  │ ──→ 触发 _clear_dependency()
                    └─────────────┘
```

**状态校验**：源码第 82-83 行硬编码了合法状态列表：

```python
if status not in ("pending", "in_progress", "completed"):
    raise ValueError(f"Invalid status: {status}")
```

注意：源码中没有 `cancelled` 状态。这是一个精简设计——生产系统可能需要更多状态（如 `blocked`、`cancelled`、`failed`），但教学版本只保留核心三态。

---

## 3. TaskManager 源码分析

### 3.1 初始化与 ID 分配

```python
# 源码 s07_task_system.py 第 46-53 行
class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(exist_ok=True)           # 确保 .tasks/ 目录存在
        self._next_id = self._max_id() + 1      # 从磁盘文件中恢复最大 ID

    def _max_id(self) -> int:
        """扫描所有 task_*.json 文件，提取文件名中的 ID 数字"""
        ids = [int(f.stem.split("_")[1]) for f in self.dir.glob("task_*.json")]
        return max(ids) if ids else 0
```

**关键细节**：

1. `_max_id()` 从文件名解析 ID（`task_1.json` -> `1`），而非读取文件内容。这是高效的——避免读取所有文件内容。
2. `_next_id` 在内存中递增，重启后通过 `_max_id()` 恢复。即使中间有任务被删除（ID 不连续），`max(ids) + 1` 也能保证新 ID 不冲突。

### 3.2 加载与保存

```python
# 源码 s07_task_system.py 第 56-64 行
def _load(self, task_id: int) -> dict:
    path = self.dir / f"task_{task_id}.json"
    if not path.exists():
        raise ValueError(f"Task {task_id} not found")
    return json.loads(path.read_text())

def _save(self, task: dict):
    path = self.dir / f"task_{task['id']}.json"
    path.write_text(json.dumps(task, indent=2))
```

**每个任务一个文件的设计理由**：

| 理由 | 说明 |
|------|------|
| **原子性** | 更新任务 #2 的状态时，只写入 `task_2.json`，不影响其他任务文件。即使写入中途崩溃，损坏的只有一个文件 |
| **并发友好** | 多个 Agent（或线程）可以同时更新不同任务，文件系统天然提供文件级隔离，无需额外的锁机制 |
| **简单恢复** | 损坏一个文件不影响其他任务。可以单独修复或删除损坏的 `task_N.json` |
| **按需读取** | 只需操作某个任务时，只读写该文件，无需加载全部任务到内存 |

**替代方案对比**：如果把所有任务存在一个 `tasks.json` 文件中，读写更简单，但并发时需要加锁，且一个文件损坏就丢失所有任务。

### 3.3 创建任务

```python
# 源码 s07_task_system.py 第 66-73 行
def create(self, subject: str, description: str = "") -> str:
    task = {
        "id": self._next_id, "subject": subject, "description": description,
        "status": "pending", "blockedBy": [], "blocks": [], "owner": "",
    }
    self._save(task)
    self._next_id += 1
    return json.dumps(task, indent=2)
```

**注意**：创建时 `blockedBy` 和 `blocks` 都是空列表。依赖关系通过 `update()` 的 `add_blocked_by` 和 `add_blocks` 参数单独建立。

### 3.4 更新任务（核心逻辑）

这是 `TaskManager` 最复杂的方法。逐段分析源码（第 78-102 行）：

```python
def update(self, task_id: int, status: str = None,
           add_blocked_by: list = None, add_blocks: list = None) -> str:
    task = self._load(task_id)

    # ---- 1. 状态更新 ----
    if status:
        if status not in ("pending", "in_progress", "completed"):
            raise ValueError(f"Invalid status: {status}")
        task["status"] = status
        # 状态传播：完成时自动解除所有下游的阻塞
        if status == "completed":
            self._clear_dependency(task_id)

    # ---- 2. 添加被阻塞关系 ----
    if add_blocked_by:
        task["blockedBy"] = list(set(task["blockedBy"] + add_blocked_by))
        #               ^^^^ set() 去重，防止重复添加同一依赖

    # ---- 3. 添加阻塞关系（双向维护） ----
    if add_blocks:
        task["blocks"] = list(set(task["blocks"] + add_blocks))
        # 同时更新被阻塞方的 blockedBy（反向引用维护）
        for blocked_id in add_blocks:
            try:
                blocked = self._load(blocked_id)
                if task_id not in blocked["blockedBy"]:
                    blocked["blockedBy"].append(task_id)
                    self._save(blocked)
            except ValueError:
                pass  # 被阻塞的任务不存在，静默忽略
    self._save(task)
    return json.dumps(task, indent=2)
```

**关键设计点**：

1. **`add_blocked_by` 和 `add_blocks` 都接受列表**（不是单个整数）。源码第 89 行 `task["blockedBy"] + add_blocked_by` 要求 `add_blocked_by` 是可迭代类型。
2. **双向自动维护**：当你调用 `task_update(task_id=1, addBlocks=[2])` 时，源码不仅更新 task_1 的 `blocks` 列表，还会自动更新 task_2 的 `blockedBy` 列表加入 1。这保证了双向引用的一致性。
3. **容错设计**：`try/except ValueError` 在第 99 行静默处理了被阻塞任务不存在的情况，允许先声明依赖、后创建任务的灵活用法。

### 3.5 状态传播：`_clear_dependency` 的精确逻辑

这是整个任务系统最关键的机制。当一个任务被标记为 `completed` 时，需要解除所有等待它的下游任务。

```python
# 源码 s07_task_system.py 第 104-110 行
def _clear_dependency(self, completed_id: int):
    """Remove completed_id from all other tasks' blockedBy lists."""
    for f in self.dir.glob("task_*.json"):        # 遍历所有任务文件
        task = json.loads(f.read_text())            # 读取每个任务
        if completed_id in task.get("blockedBy", []):  # 检查是否被该任务阻塞
            task["blockedBy"].remove(completed_id)      # 移除阻塞
            self._save(task)                             # 写回文件
```

**执行流程图**：

```
task_update(task_id=1, status="completed")
    │
    ├── 1. 校验 status ∈ {"pending", "in_progress", "completed"}  ✓
    ├── 2. 设置 task_1.status = "completed"
    │
    └── 3. 调用 _clear_dependency(1)
            │
            ├── glob("task_*.json") → [task_1.json, task_2.json, task_3.json, task_4.json]
            │
            ├── 读 task_1.json: blockedBy=[] → 不包含 1 → 跳过
            ├── 读 task_2.json: blockedBy=[1] → 包含 1 → 移除 → blockedBy=[] → 写回
            ├── 读 task_3.json: blockedBy=[1] → 包含 1 → 移除 → blockedBy=[] → 写回
            └── 读 task_4.json: blockedBy=[2,3] → 不包含 1 → 跳过
```

**全局扫描的权衡**：

| 方面 | 分析 |
|------|------|
| **优点** | 实现简单，不需要额外的索引结构 |
| **优点** | 天然幂等——重复调用不会出错 |
| **代价** | O(N) 复杂度，每次完成一个任务需要读取所有文件 |
| **实际影响** | 教学场景下任务数量有限（通常 < 100），性能可接受 |
| **生产优化** | 可维护一个 `dependency_index` 字典加速查找，但会增加复杂度 |

### 3.6 列出任务

```python
# 源码 s07_task_system.py 第 112-123 行
def list_all(self) -> str:
    tasks = []
    for f in sorted(self.dir.glob("task_*.json")):  # sorted 保证按文件名排序
        tasks.append(json.loads(f.read_text()))
    if not tasks:
        return "No tasks."
    lines = []
    for t in tasks:
        marker = {"pending": "[ ]", "in_progress": "[>]", "completed": "[x]"}.get(t["status"], "[?]")
        blocked = f" (blocked by: {t['blockedBy']})" if t.get("blockedBy") else ""
        lines.append(f"{marker} #{t['id']}: {t['subject']}{blocked}")
    return "\n".join(lines)
```

输出示例：

```
[x] #1: 项目初始化
[ ] #2: 认证模块
[ ] #3: 数据库模块
[ ] #4: 集成测试 (blocked by: [2, 3])
```

注意：`sorted(self.dir.glob(...))` 按文件名字典序排列。由于文件名格式为 `task_N.json`，当 N > 9 时可能出现 `task_1.json, task_10.json, task_2.json` 的非预期排序。这是当前实现的已知限制。

---

## 4. 工具集成

### 4.1 工具定义（与源码完全一致）

源码 `s07_task_system.py` 第 189-206 行定义了 8 个工具。其中 4 个是文件操作工具（bash、read_file、write_file、edit_file），4 个是任务管理工具：

```python
# 任务相关工具定义（源码第 198-206 行）
{"name": "task_create", "description": "Create a new task.",
 "input_schema": {"type": "object", "properties": {
     "subject": {"type": "string"},
     "description": {"type": "string"}
 }, "required": ["subject"]}},

{"name": "task_update", "description": "Update a task's status or dependencies.",
 "input_schema": {"type": "object", "properties": {
     "task_id": {"type": "integer"},
     "status": {"type": "string", "enum": ["pending", "in_progress", "completed"]},
     "addBlockedBy": {"type": "array", "items": {"type": "integer"}},
     "addBlocks": {"type": "array", "items": {"type": "integer"}}
 }, "required": ["task_id"]}},

{"name": "task_list", "description": "List all tasks with status summary.",
 "input_schema": {"type": "object", "properties": {}}},

{"name": "task_get", "description": "Get full details of a task by ID.",
 "input_schema": {"type": "object", "properties": {
     "task_id": {"type": "integer"}
 }, "required": ["task_id"]}},
```

### 4.2 注册处理器

```python
# 源码 s07_task_system.py 第 178-187 行
TOOL_HANDLERS = {
    "bash":        lambda **kw: run_bash(kw["command"]),
    "read_file":   lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file":  lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":   lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "task_create": lambda **kw: TASKS.create(kw["subject"], kw.get("description", "")),
    "task_update": lambda **kw: TASKS.update(
        kw["task_id"], kw.get("status"),
        kw.get("addBlockedBy"),     # 注意：JSON 的 camelCase 映射
        kw.get("addBlocks")
    ),
    "task_list":   lambda **kw: TASKS.list_all(),
    "task_get":    lambda **kw: TASKS.get(kw["task_id"]),
}
```

**命名约定**：工具 schema 使用 camelCase（`addBlockedBy`、`addBlocks`），Python 代码内部使用 snake_case（`add_blocked_by`、`add_blocks`）。Lambda 层负责映射。

---

## 5. Agent 循环中的集成

### 5.1 循环结构

源码的 `agent_loop`（第 209-228 行）与前面章节的核心结构完全一致：

```python
def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                try:
                    output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                except Exception as e:
                    output = f"Error: {e}"
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(output)})
        messages.append({"role": "user", "content": results})
```

**s07 没有修改循环本身**——它只是通过扩展 `TOOLS` 和 `TOOL_HANDLERS` 来增加能力。这体现了"渐进增强"的设计原则。

### 5.2 System Prompt

```python
SYSTEM = f"You are a coding agent at {WORKDIR}. Use task tools to plan and track work."
```

System prompt 直接提示 LLM 使用任务工具来规划和跟踪工作。LLM 根据用户意图自主决定何时创建任务、何时建立依赖、何时更新状态。

---

## 6. 完整示例

### 6.1 创建任务图

```
用户：创建一个登录功能的任务计划

Agent（可能的工具调用序列）：
  task_create(subject="设计数据库 schema")
  → {"id": 1, "subject": "设计数据库 schema", "status": "pending", "blockedBy": [], "blocks": [], ...}

  task_create(subject="实现用户模型")
  → {"id": 2, ...}

  task_create(subject="实现认证 API")
  → {"id": 3, ...}

  task_create(subject="编写登录页面")
  → {"id": 4, ...}

  task_create(subject="编写集成测试")
  → {"id": 5, ...}

  # 设置依赖关系
  task_update(task_id=2, addBlockedBy=[1])   # 用户模型等待 schema
  task_update(task_id=3, addBlockedBy=[2])   # API 等待用户模型
  task_update(task_id=4, addBlockedBy=[3])   # 页面等待 API
  task_update(task_id=5, addBlockedBy=[3, 4])  # 测试等待 API 和页面

  task_list()
```

### 6.2 任务列表输出

```
[ ] #1: 设计数据库 schema
[ ] #2: 实现用户模型 (blocked by: [1])
[ ] #3: 实现认证 API (blocked by: [2])
[ ] #4: 编写登录页面 (blocked by: [3])
[ ] #5: 编写集成测试 (blocked by: [3, 4])
```

### 6.3 执行与状态传播

```
# 开始任务 1
task_update(task_id=1, status="in_progress")
# ... Agent 使用 bash/write_file 等工具完成 schema 设计 ...
task_update(task_id=1, status="completed")
# _clear_dependency(1) 被调用
# → 扫描所有 task_*.json，发现 task_2 的 blockedBy 包含 1
# → 从 task_2.blockedBy 中移除 1
# task_2 现在可执行了

# 开始任务 2
task_update(task_id=2, status="in_progress")
# ... 完成用户模型 ...
task_update(task_id=2, status="completed")
# _clear_dependency(2) → 解除 task_3 的阻塞

# 开始任务 3
task_update(task_id=3, status="in_progress")
# ... 完成 API ...
task_update(task_id=3, status="completed")
# _clear_dependency(3) → 同时解除 task_4 和 task_5 的部分阻塞
# task_4 的 blockedBy 从 [3] 变为 []
# task_5 的 blockedBy 从 [3, 4] 变为 [4]（还需等待 task_4）

# task_4 现在可执行
task_update(task_id=4, status="in_progress")
# ... 完成登录页面 ...
task_update(task_id=4, status="completed")
# _clear_dependency(4) → 解除 task_5 的最后阻塞
# task_5 的 blockedBy 从 [4] 变为 []

# task_5 现在可执行
```

---

## 7. 架构分析

### 7.1 与 s03 的对比

| 特性 | TodoManager (s03) | TaskManager (s07) |
|------|-------------------|-------------------|
| 存储 | 内存中的 Python 列表 | 磁盘上的 JSON 文件目录 |
| 持久化 | 无，上下文压缩后丢失 | 有，`.tasks/` 目录跨会话存活 |
| 依赖关系 | 无 | `blockedBy` + `blocks` 双向引用 |
| 合法状态 | `pending` / `in_progress` / `completed` | `pending` / `in_progress` / `completed` |
| 状态传播 | 无（手动管理） | 自动（`_clear_dependency` 全局扫描） |
| 所有权 | 无 | `owner` 字段（为多 Agent 预留） |
| ID 生成 | 列表索引（连续） | 文件名解析（不连续但唯一） |

### 7.2 文件系统作为数据库

s07 本质上是用文件系统模拟了一个简单的数据库：

| 数据库概念 | s07 对应 | 说明 |
|-----------|---------|------|
| 表 | `.tasks/` 目录 | 所有任务文件的容器 |
| 行 | `task_N.json` | 每个文件是一条记录 |
| 主键 | `id`（文件名编码） | 通过文件名直接定位 |
| 索引 | 无 | 需要扫描全部文件（`glob`） |
| 事务 | 文件级原子写入 | `write_text()` 覆盖写入 |
| 查询 | `glob` + `json.loads` | 全表扫描模式 |

这种设计在教学和轻量级场景中足够用。生产环境会使用真正的数据库（如 SQLite）来获得索引和事务支持。

---

## 8. 使用场景

### 8.1 适合任务系统的场景

- **多步骤开发流程**：设计 -> 实现 -> 测试 -> 部署，每一步依赖前一步的产出
- **跨会话项目**：今天做不完，明天继续。任务状态持久化在磁盘
- **复杂重构**：拆分为多个子任务，建立依赖关系，确保按正确顺序执行
- **多 Agent 协作**：`owner` 字段预留了任务认领机制（s11 会用到）

### 8.2 不适合的场景

- **单步操作**：只需执行一个命令，无需任务管理
- **实时性要求高**：文件 I/O 有延迟，不适合毫秒级操作
- **海量任务**：全局扫描模式在任务数量 > 1000 时会有性能问题

---

## 9. 常见问题

### Q1：为什么每个任务一个文件，而不是一个大 JSON 文件？

**答案**：三个原因——原子性、并发友好、故障隔离。

如果所有任务存在一个 `tasks.json` 中：
- 更新任何一个任务都需要读写整个文件
- 两个 Agent 同时更新不同任务会互相覆盖（需要文件锁）
- 文件损坏会丢失所有任务数据

每个任务独立文件则：
- 更新任务 #2 只涉及 `task_2.json`
- 不同任务的写入操作天然互不影响
- 一个文件损坏只影响该任务

### Q2：`_clear_dependency` 为什么要扫描所有文件？

**答案**：因为没有反向索引。

当前设计中，"哪些任务被 task_1 阻塞"这个信息分散在各个文件的 `blockedBy` 字段中。要知道哪些任务需要解除阻塞，唯一的办法就是遍历所有文件。

更高效的方案是维护一个 `blocks_index: Dict[int, List[int]]` 字典，记录"task_1 阻塞了哪些任务"。但这增加了维护复杂度，在任务数量有限的教学场景中不必要。

### Q3：如何防止循环依赖？

当前源码没有实现循环依赖检测。如果用户创建 A 依赖 B、B 依赖 A 的循环，会导致任务永远无法变为可执行状态。

这是一个合理的简化——在实践中，Agent 通常按合理的逻辑顺序建立依赖。如果需要防护，可以在 `update()` 中添加 DFS 环检测：

```python
def _check_cycle(self, from_id: int, to_id: int) -> bool:
    """检查添加 from_id -> to_id 依赖是否会形成环"""
    visited = set()
    def dfs(current):
        if current == from_id:
            return True  # 找到环
        if current in visited:
            return False
        visited.add(current)
        task = self._load(current)
        for dep in task.get("blockedBy", []):
            if dfs(dep):
                return True
        return False
    return dfs(to_id)
```

### Q4：ID 不连续会有问题吗？

不会。ID 的唯一作用是生成文件名和查找文件。即使删除了 `task_3.json` 导致 ID 变成 `1, 2, 4, 5`，系统仍然正常工作。`_max_id()` 取最大值 + 1 保证了新 ID 不会与现有 ID 冲突。

### Q5：task_*.json 文件损坏（如写入中途断电），如何手动修复？

由于每个任务独立存储为一个 JSON 文件，损坏通常只影响单个文件。修复步骤如下：

**步骤 1：定位损坏文件**

```bash
# 遍历所有任务文件，检查 JSON 是否合法
for f in .tasks/task_*.json; do
    python -c "import json; json.load(open('$f'))" 2>/dev/null || echo "CORRUPTED: $f"
done
```

**步骤 2：手动修复**

打开损坏的文件，根据 JSON 语法错误进行修复。常见情况：
- 文件截断（写到一半断电）：补全缺失的 `}` 或 `]`
- 编码异常：删除文件中的乱码字符

**步骤 3：如果无法修复，重建任务**

```bash
# 删除损坏的文件
rm .tasks/task_3.json

# 通过 Agent 重新创建任务
# 注意：新任务的 ID 会递增（不会复用旧 ID），依赖关系中引用旧 ID 的条目需要手动更新
```

**预防措施**：生产环境中可以使用"写入临时文件 + 原子重命名"策略来避免写入中断导致的数据损坏：

```python
import tempfile, os

def _save(self, task: dict):
    path = self.dir / f"task_{task['id']}.json"
    # 先写入临时文件
    fd, tmp = tempfile.mkstemp(dir=self.dir, suffix=".tmp")
    with os.fdopen(fd, "w") as f:
        f.write(json.dumps(task, indent=2))
    # 原子重命名（rename 在同一文件系统上是原子操作）
    os.replace(tmp, path)
```

---

## 10. 最佳实践

### 10.1 任务粒度

```
太粗：
  "实现登录功能" → 无法跟踪进度，一个任务可能要几小时

太细：
  "创建文件 auth.py"
  "定义 User 类"
  "实现 login 方法"
  "添加参数验证"
  → 管理复杂度超过实际工作

合适：
  "设计数据库 schema"
  "实现用户模型"
  "实现认证 API"
  "编写登录页面"
  "编写集成测试"
```

### 10.2 依赖设计原则

```
好的依赖：
  - 逻辑依赖：B 真正需要 A 的产出（如 API 测试依赖 API 实现）
  - 明确声明：通过 addBlockedBy 显式表达
  - 最小化：只依赖真正需要的任务

坏的依赖：
  - 顺序偏好：A 应该在 B 之前做（但 B 实际不依赖 A 的产出）
  - 过度依赖：所有任务都依赖第一个任务
```

### 10.3 任务描述规范

```python
# 好的任务描述
task_create(
    subject="实现用户认证 API",
    description="实现 /api/login 和 /api/logout 端点，使用 JWT 认证"
)

# 差的任务描述
task_create(subject="API", description="做 API")
```

---

## 11. 小结

### 11.1 核心要点

| 要点 | 说明 |
|------|------|
| **持久化** | 任务存储在 `.tasks/` 目录下的 JSON 文件中，跨会话存活 |
| **DAG** | 通过 `blockedBy` + `blocks` 双向引用表达有向无环依赖关系 |
| **状态传播** | `_clear_dependency()` 通过全局扫描自动解除下游阻塞 |
| **原子操作** | 每个任务一个文件，更新互不影响 |
| **渐进增强** | 只扩展工具集，不修改 Agent 循环本身 |

### 11.2 代码模板

```python
# 创建任务
task_create(subject="任务名称", description="详细描述")

# 设置依赖（双向自动维护）
task_update(task_id=2, addBlockedBy=[1])

# 开始执行
task_update(task_id=1, status="in_progress")

# 完成任务（自动解除下游阻塞）
task_update(task_id=1, status="completed")

# 查看所有任务
task_list()

# 查看单个任务详情
task_get(task_id=1)
```

### 11.3 关键洞察

```
任务系统的核心价值：
1. 将复杂目标分解为可管理的单元
2. 用 DAG 明确任务间的依赖关系
3. 通过状态传播自动管理可执行性
4. 持久化到磁盘，在上下文压缩后仍可恢复

为什么这个设计能工作：
- 文件系统提供了天然的原子性和隔离性
- 全局扫描虽然 O(N)，但在小规模下足够快
- 双向引用让依赖查询和状态传播都变得简单
- LLM 只需关注"做什么"，依赖管理完全自动化
```

---

## 12. 练习

### 练习 1：任务优先级

在任务对象中添加 `priority` 字段（high/medium/low）。修改 `list_all()` 的输出格式，在任务描述前显示优先级标记。

**提示**：修改 `create()` 方法的参数和 `list_all()` 的格式化逻辑。

### 练习 2：循环依赖检测

在 `update()` 方法中添加环检测。当 `addBlockedBy` 或 `addBlocks` 操作可能创建循环时，抛出异常。

**提示**：使用 DFS 遍历依赖图，检查新依赖是否会导致从目标节点回到源节点的路径。

### 练习 3：发现排序问题

在 Agent 中创建 12 个任务，然后调用 `task_list`。仔细观察输出中的任务编号顺序——当 ID 超过 9 时，排序结果是否符合你的预期？如果不正确，思考一下根因是什么（提示：`sorted()` 默认按什么规则排序？文件名 `task_10.json` 和 `task_2.json` 谁排在前面？），然后尝试修复 `list_all()` 方法中的排序逻辑。

### 练习 4：依赖可视化

添加一个 `task_dot` 工具，生成任务依赖图的 Graphviz DOT 格式输出。

**提示**：遍历所有任务，为每个 `blockedBy` 关系生成 `task_A -> task_B` 的 DOT 边。

---

## 13. 下一步

现在任务系统可以持久化到磁盘了。但执行长时间任务（如运行测试）仍然会阻塞 Agent 的思考。

在 [s08：后台任务](./s08-background-tasks.md) 中，我们将学习：
- 如何在守护线程中异步运行子进程
- 通知队列的 drain 模式如何在主线程和后台线程间安全传递结果
- 超时保护机制防止任务无限运行
- 后台任务与 s07 任务系统如何协同工作

---

< 上一章：[s06-上下文压缩](./s06-context-compact.md) | [目录](./index.md) | 下一章：[s08-后台任务](./s08-background-tasks.md) >
