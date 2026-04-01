# s08：后台任务 —— 并发执行的艺术

> **核心口号**：*"慢操作后台运行，Agent 继续思考"*

> **关键洞察**：*"Fire and forget -- the agent doesn't block while the command runs."*

---

## 学习目标

完成本章后，你将能够：

1. **理解阻塞问题** —— 同步执行长时间操作为什么会拖慢 Agent
2. **掌握守护线程机制** —— `threading.Thread(daemon=True)` 的实际行为
3. **理解通知队列的 drain 模式** —— 主线程如何在 LLM 调用前安全收集结果
4. **分析 threading.Lock 的使用** —— 保护共享状态的具体范围
5. **理解超时保护** —— `timeout=300` 的实际值与子进程超时机制

---

## 0. 上手演练（建议先做）

先运行后台任务示例，再理解并发架构：

```bash
uv run python agents/s08_background_tasks.py
```

建议观察：

1. 在 Agent 中输入"后台运行一个 sleep 5 命令"，观察 `background_run` 返回 task_id 的速度。
2. 立即输入其他命令，验证主循环没有被阻塞。
3. 5 秒后输入任意查询，观察 `<background-results>` 如何被注入到上下文中。
4. 尝试运行一个耗时超过 300 秒的命令（如 `sleep 400`），观察超时保护如何触发。

### 本章完成标准

- 能解释"阻塞执行"与"后台执行"在吞吐上的差异。
- 能独立实现后台任务的创建、查询、通知回收与超时保护。
- 能画出主线程和后台线程的交互时序图，标注 `threading.Lock` 的保护范围。
- 能评估一个命令是否适合放入后台并给出理由。

---

## 1. 问题的本质

### 1.1 阻塞式执行的问题

回顾 s02 中的 `run_bash`（同步版本）：

```python
# s02 中的同步执行
def run_bash(command: str) -> str:
    r = subprocess.run(command, shell=True, cwd=WORKDIR,
                       capture_output=True, text=True, timeout=120)
    # ↑ Agent 循环在此等待，直到命令完成
```

**问题场景**：

```
用户：安装依赖并创建配置文件

Agent（同步执行）：
  1. run_bash("pip install -r requirements.txt")  ← 等待 120 秒
     Agent 循环被阻塞，无法处理任何其他操作
     ↓
  2. write_file("config.yaml", "...")              ← 等待 0.01 秒
     ↓
  3. 返回结果

总时间：~120 秒（串行）
```

在这 120 秒内，Agent 完全无法思考、无法响应用户、无法执行任何工具。

### 1.2 后台执行的解法

```
用户：安装依赖并创建配置文件

Agent（后台执行）：
  1. background_run("pip install -r requirements.txt")
     → 立即返回："Background task a1b2c3d4 started"
     后台线程开始运行 pip install
     ↓
  2. write_file("config.yaml", "...")              ← 立即执行，无需等待
     ↓
  3. 继续其他工作...
     ↓
  4. 下一轮 LLM 调用前，drain_notifications() 检查后台结果
     如果 pip install 已完成，结果被注入上下文
     ↓
  5. Agent 处理安装结果并继续

总时间：~120 秒（但 Agent 在等待期间做了其他事）
pip install 和配置文件创建并行执行
```

---

## 2. 后台执行架构

### 2.1 架构图

```
主线程 (Agent 循环)                     后台线程 1               后台线程 2
┌──────────────────────┐               ┌──────────────┐        ┌──────────────┐
│                      │               │ pip install  │        │   pytest     │
│  user input          │               │              │        │              │
│      ↓               │    spawn ───→ │  运行中...   │        │              │
│  agent_loop()        │               │      ↓       │        │              │
│      ↓               │               │  subprocess  │        │              │
│  drain_notifications │◄── enqueue ───│  完成        │        │  运行中...   │
│      ↓               │    (加锁)     │              │        │      ↓       │
│  注入 <background-   │               └──────────────┘        │  subprocess  │
│    results> 到消息   │                                       │      ↓       │
│      ↓               │               ┌──────────────┐        │  完成        │
│  LLM 调用            │               │ _notification│◄── enqueue ──│       │
│  (带新上下文)         │               │  _queue      │        │              │
│      ↓               │               │  [结果1,     │        └──────────────┘
│  执行工具...         │               │   结果2]     │
│      ↓               │               └──────────────┘
│  drain_notifications │── 取出并清空队列
│      ↓               │
│  LLM 调用            │
└──────────────────────┘
```

### 2.2 时间线对比

```
同步执行（s02 的 run_bash）：
├── pip install (120s) ──┤├── 创建配置 (5s) ──┤├── 其他工作 (10s) ──┤
────────────────────────────────────────────────────────────────────→ 135s

异步执行（s08 的 background_run）：
├── pip install (120s) ──────────────────┤    ← 后台线程
├── 创建配置 (5s) ─┤├── 其他工作 (10s) ─┤       ← 主线程
──────────────────────────────────────────────→ 120s (并行)

Agent 空闲等待时间：同步 125s vs 异步 0s
```

---

## 3. BackgroundManager 源码分析

### 3.1 核心数据结构

```python
# 源码 s08_background_tasks.py 第 49-53 行
class BackgroundManager:
    def __init__(self):
        self.tasks = {}                    # task_id -> {status, result, command}
        self._notification_queue = []      # 完成任务的结果列表
        self._lock = threading.Lock()      # 保护 _notification_queue 的互斥锁
```

**三个数据成员的职责**：

| 成员 | 类型 | 作用 | 访问线程 |
|------|------|------|----------|
| `self.tasks` | `dict` | 存储所有后台任务的状态和结果 | 主线程写（run），后台线程写（_execute），主线程读（check） |
| `self._notification_queue` | `list` | 待投递的完成通知 | 后台线程写（_execute），主线程读写（drain） |
| `self._lock` | `threading.Lock` | 保护 `_notification_queue` 的互斥访问 | 所有线程 |

**注意**：`self.tasks` 字典的访问没有被锁保护。在当前实现中这是可以接受的，因为：
- `run()` 在后台线程启动前写入（主线程）
- `_execute()` 在线程启动后才修改对应 task_id 的条目（后台线程）
- `check()` 只读取（主线程）

这种"谁先写、谁后写"的时序保证替代了显式加锁，是一种简化的并发策略。

### 3.2 启动后台任务

```python
# 源码 s08_background_tasks.py 第 55-63 行
def run(self, command: str) -> str:
    """Start a background thread, return task_id immediately."""
    task_id = str(uuid.uuid4())[:8]  # 取 UUID 前 8 字符作为短 ID
    self.tasks[task_id] = {"status": "running", "result": None, "command": command}
    thread = threading.Thread(
        target=self._execute,
        args=(task_id, command),
        daemon=True  # 关键：守护线程
    )
    thread.start()
    return f"Background task {task_id} started: {command[:80]}"
```

**守护线程（daemon=True）的含义**：

| 属性 | 守护线程 | 非守护线程 |
|------|---------|-----------|
| 主程序退出时 | 自动终止 | 阻止程序退出，直到线程完成 |
| 适用场景 | 后台辅助任务 | 必须完成的任务 |
| 风险 | 任务可能被中途终止 | 程序可能无法退出 |

在 Agent 场景中选择 `daemon=True` 是正确的：后台命令的结果是辅助性的，不应该阻止用户退出程序。

**task_id 的生成**：`str(uuid.uuid4())[:8]` 取 UUID4 的前 8 个十六进制字符（如 `"a1b2c3d4"`）。8 个十六进制字符提供 4,294,967,296 种可能，在单次会话中碰撞概率极低。

### 3.3 后台线程执行（核心逻辑）

```python
# 源码 s08_background_tasks.py 第 65-88 行
def _execute(self, task_id: str, command: str):
    """Thread target: run subprocess, capture output, push to queue."""
    try:
        r = subprocess.run(
            command, shell=True, cwd=WORKDIR,
            capture_output=True, text=True, timeout=300  # 300 秒超时
        )
        output = (r.stdout + r.stderr).strip()[:50000]   # 截断到 50000 字符
        status = "completed"
    except subprocess.TimeoutExpired:
        output = "Error: Timeout (300s)"
        status = "timeout"
    except Exception as e:
        output = f"Error: {e}"
        status = "error"

    # 更新任务状态（无锁，见 3.1 的分析）
    self.tasks[task_id]["status"] = status
    self.tasks[task_id]["result"] = output or "(no output)"

    # 将结果加入通知队列（加锁保护）
    with self._lock:
        self._notification_queue.append({
            "task_id": task_id,
            "status": status,
            "command": command[:80],
            "result": (output or "(no output)")[:500],  # 通知中只保留前 500 字符
        })
```

**执行流程图**：

```
_execute(task_id, command)
    │
    ├── subprocess.run(command, timeout=300)
    │       │
    │       ├── 正常完成 → output = stdout+stderr, status = "completed"
    │       ├── TimeoutExpired → output = "Error: Timeout (300s)", status = "timeout"
    │       └── 其他异常 → output = error message, status = "error"
    │
    ├── self.tasks[task_id]["status"] = status    ← 更新任务状态
    ├── self.tasks[task_id]["result"] = output    ← 保存完整输出
    │
    └── with self._lock:                          ← 加锁保护队列
            self._notification_queue.append(...)   ← 追加通知
```

**超时保护分析**：

| 超时值 | 对比 | 说明 |
|--------|------|------|
| `timeout=300`（后台任务） | 300 秒 = 5 分钟 | 源码第 70 行，适用于长时间构建/测试 |
| `timeout=120`（同步 bash） | 120 秒 = 2 分钟 | 源码第 126 行，同步场景超时更短 |
| `timeout=120`（同步 bash in s07） | 120 秒 = 2 分钟 | 同上 |

后台任务允许更长的超时（300s vs 120s），因为它不阻塞主循环。即使命令运行 5 分钟，Agent 仍然可以继续思考和执行其他操作。

**输出截断策略**：

| 位置 | 截断值 | 原因 |
|------|--------|------|
| `self.tasks[task_id]["result"]` | 50000 字符 | 完整保存，供 `check()` 查看 |
| `self._notification_queue` 中的 `result` | 500 字符 | 注入 LLM 上下文时节省 token |
| `command` 显示 | 80 字符 | 界面显示空间有限 |

### 3.4 threading.Lock 的精确分析

```python
# 锁的定义
self._lock = threading.Lock()

# 写入端（后台线程，_execute 方法第 82-88 行）
with self._lock:
    self._notification_queue.append({...})

# 读取端（主线程，drain_notifications 方法第 102-107 行）
with self._lock:
    notifs = list(self._notification_queue)   # 复制
    self._notification_queue.clear()           # 清空
return notifs
```

**锁保护的范围**：只保护 `self._notification_queue`，不保护 `self.tasks`。

**为什么需要锁**：`_notification_queue` 是一个 Python `list`，虽然 `append()` 在 CPython 中是线程安全的（GIL 保证），但 `drain_notifications()` 中的"复制 + 清空"不是原子操作。如果后台线程在 `list()` 和 `clear()` 之间执行了 `append()`，那个通知就会丢失。锁确保了"取出所有通知"是一个原子操作。

**drain 模式**：

```
drain_notifications() 的语义：取出并清空

调用前：_notification_queue = [notif_A, notif_B, notif_C]
调用中：with lock: copy → [notif_A, notif_B, notif_C], clear → []
调用后：返回 [notif_A, notif_B, notif_C]，队列已清空

这保证：
1. 每个通知只被处理一次（取出后清空）
2. 不会遗漏（加锁期间新完成的通知会在下一轮被取出）
3. 不会重复（clear 后队列为空）
```

### 3.5 查询任务状态

```python
# 源码 s08_background_tasks.py 第 90-100 行
def check(self, task_id: str = None) -> str:
    """Check status of one task or list all."""
    if task_id:
        t = self.tasks.get(task_id)
        if not t:
            return f"Error: Unknown task {task_id}"
        return f"[{t['status']}] {t['command'][:60]}\n{t.get('result') or '(running)'}"
    # task_id 为 None 时列出所有任务
    lines = []
    for tid, t in self.tasks.items():
        lines.append(f"{tid}: [{t['status']}] {t['command'][:60]}")
    return "\n".join(lines) if lines else "No background tasks."
```

**`check()` 的双重功能**：

| 调用方式 | 行为 |
|----------|------|
| `check(task_id="a1b2c3d4")` | 返回单个任务的详细状态和结果 |
| `check()` 或 `check(task_id=None)` | 列出所有后台任务的状态摘要 |

注意：当任务仍在运行时，`t.get('result')` 返回 `None`，显示为 `"(running)"`。

---

## 4. 集成到 Agent 循环

### 4.1 工具定义（与源码完全一致）

源码 `s08_background_tasks.py` 第 171-184 行定义了 6 个工具。其中 4 个是文件操作（与 s07 相同），2 个是后台任务专用：

```python
# 后台任务工具定义（源码第 180-183 行）
{"name": "background_run",
 "description": "Run command in background thread. Returns task_id immediately.",
 "input_schema": {"type": "object", "properties": {
     "command": {"type": "string"}
 }, "required": ["command"]}},

{"name": "check_background",
 "description": "Check background task status. Omit task_id to list all.",
 "input_schema": {"type": "object", "properties": {
     "task_id": {"type": "string"}
 }}},
```

**注意工具命名**：`check_background`（而非 `background_check`）。这遵循"动词_名词"的命名模式。

### 4.2 注册处理器

```python
# 源码 s08_background_tasks.py 第 162-169 行
TOOL_HANDLERS = {
    "bash":             lambda **kw: run_bash(kw["command"]),
    "read_file":        lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file":       lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":        lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "background_run":   lambda **kw: BG.run(kw["command"]),
    "check_background": lambda **kw: BG.check(kw.get("task_id")),
}
```

### 4.3 Agent 循环中的通知注入（核心集成点）

这是 s08 对 Agent 循环的唯一修改——在每次 LLM 调用前 drain 通知队列：

```python
# 源码 s08_background_tasks.py 第 187-214 行
def agent_loop(messages: list):
    while True:
        # ===== s08 新增：在 LLM 调用前 drain 通知队列 =====
        notifs = BG.drain_notifications()
        if notifs and messages:
            notif_text = "\n".join(
                f"[bg:{n['task_id']}] {n['status']}: {n['result']}" for n in notifs
            )
            # 注入为 user 消息（因为 LLM 要求 user/assistant 交替）
            messages.append({"role": "user",
                "content": f"<background-results>\n{notif_text}\n</background-results>"})
            messages.append({"role": "assistant",
                "content": "Noted background results."})
        # ===== 新增逻辑结束 =====

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

**通知注入的时序分析**：

```
时间线：

t=0s    用户输入 "运行 pytest 并创建新测试文件"
t=0s    Agent 调用 background_run("pytest tests/")
        → 返回 "Background task a1b2c3 started"，后台线程启动
t=0.1s  Agent 调用 write_file("tests/test_new.py", "...")
        → 返回 "Wrote 1234 bytes"
t=0.2s  LLM 返回 end_turn（本轮无更多工具调用）
        → Agent 回复用户

t=0.2s  用户输入 "检查测试结果"
t=0.2s  agent_loop 开始，drain_notifications() 被调用
        → 如果 pytest 仍在运行，队列为空，无通知注入
        → LLM 正常调用
t=2.3s  后台线程：pytest 完成，结果被 append 到 _notification_queue
        （此时 LLM 可能在处理中）

t=5s    用户输入 "测试跑完了吗？"
t=5s    agent_loop 开始，drain_notifications() 被调用
        → 队列中有 pytest 结果，注入 <background-results>
        → LLM 收到通知，可以分析测试结果
```

**消息注入格式**：

```
user: <background-results>
      [bg:a1b2c3d4] completed: 15 passed in 2.34s
      </background-results>
assistant: Noted background results.
```

`user` + `assistant` 配对是为了满足 Anthropic API 的消息交替要求。

**为什么需要同时添加 user 和 assistant 两条消息？**

Anthropic 的 Messages API 要求消息按 `user` -> `assistant` -> `user` -> `assistant` 的严格交替顺序排列。如果连续出现两条相同 role 的消息，API 会返回错误。因此注入后台通知时，必须同时添加：

1. **user 消息**：携带实际的 `<background-results>` 内容
2. **assistant 消息**：一条简短的确认（"Noted background results."），为下一轮 user 消息"垫位"

如果只添加 user 消息而不添加 assistant 消息，后续 Agent 循环中产生的 assistant 响应（LLM 的回复）就会紧跟在注入的 user 消息之后，与正常的消息流产生连续两条 user 消息的情况，导致 API 调用失败。

```
错误示范（只有 user）：
  ... → [注入 user: background-results] → [下一轮 LLM 返回 assistant] ← 可能与上方的 user 消息之间间隔正常
  但如果有其他工具结果也作为 user 消息追加，就会出现连续两条 user 消息

正确做法（user + assistant 配对）：
  ... → [注入 user: background-results] → [注入 assistant: "Noted"] → [后续正常消息流]
```

---

## 5. 完整示例

### 5.1 并发执行多个任务

```
用户：运行测试并创建新的测试文件

Agent：
  1. background_run(command="pytest tests/")
     → "Background task a1b2c3d4 started: pytest tests/"

  2. write_file(path="tests/test_new.py", content="...")
     → "Wrote 1234 bytes"

  3. LLM 返回：我已经在后台启动了测试，同时创建了新的测试文件。
     等测试结果出来后我会分析。

  ...（pytest 在后台运行，Agent 可以继续响应其他请求）...

  4. 下一轮 agent_loop 中：
     <background-results>
     [bg:a1b2c3d4] completed: === 15 passed in 2.34s ===
     </background-results>

  5. LLM：测试全部通过！15 个测试用例都成功了。
```

### 5.2 超时场景

```
用户：后台运行一个很慢的构建命令

Agent：
  background_run(command="make -j4 all")
  → "Background task e5f6g7h8 started: make -j4 all"

  ...（300 秒后）...

  下一轮 agent_loop 中：
  <background-results>
  [bg:e5f6g7h8] timeout: Error: Timeout (300s)
  </background-results>

  LLM：构建命令在 300 秒后超时了。这可能是因为项目太大或构建配置有问题。
```

### 5.3 查询正在运行的任务

```
用户：安装到哪了？

Agent：
  check_background(task_id="def45678")
  → "[running] pip install -r requirements.txt
     (running)"

  LLM：安装还在进行中。我稍后会通知你结果。
```

---

## 6. 与 s07 任务系统的协同工作

### 6.1 定位差异

s07 和 s08 解决的是不同层面的问题：

| 维度 | s07 TaskManager | s08 BackgroundManager |
|------|----------------|----------------------|
| **管理对象** | 逻辑任务（设计、实现、测试） | 系统命令（pip install、pytest） |
| **存储方式** | 磁盘文件（`.tasks/task_N.json`） | 内存字典（`self.tasks`） |
| **生命周期** | 跨会话持久存在 | 随进程结束而消失 |
| **执行方式** | Agent 主动推进状态 | 后台线程自动完成 |
| **状态模型** | `pending` → `in_progress` → `completed` | `running` → `completed`/`timeout`/`error` |
| **依赖关系** | DAG 依赖图 | 无依赖 |
| **通信机制** | 直接返回 JSON | 通知队列 |

### 6.2 组合使用场景

在实际 Agent 工作中，s07 和 s08 经常配合使用：

```
场景：Agent 需要完成"运行所有测试"这个任务

s07 层面：
  task_update(task_id=5, status="in_progress")  # 标记任务开始

s08 层面：
  background_run(command="pytest tests/")        # 实际执行测试命令
  → 立即返回，不阻塞 Agent

当后台任务完成（通知注入后）：
  task_update(task_id=5, status="completed")     # 标记任务完成
```

### 6.3 对比表

| 方面 | s07 任务系统 | s08 后台任务 |
|------|-------------|-------------|
| 执行方式 | Agent 决定何时推进 | 后台线程自动执行 |
| 存储 | 文件系统（持久化） | 内存（临时性） |
| 状态管理 | 三态（pending/in_progress/completed） | 四态（running/completed/timeout/error） |
| 适用场景 | 逻辑任务规划和跟踪 | 耗时命令的异步执行 |
| 通知机制 | 无（直接返回结果） | 通知队列 + drain 模式 |
| 依赖管理 | DAG 依赖图 | 无 |
| 核心价值 | 跨会话状态保持 | 并发执行不阻塞 |

---

## 7. 常见问题

### Q1：为什么使用守护线程（daemon=True）？

```python
# 源码第 60 行
thread = threading.Thread(target=self._execute, args=(...), daemon=True)
```

**守护线程的行为**：
- 当主线程退出时，所有守护线程自动终止
- 非守护线程会阻止 Python 进程退出

**选择守护线程的理由**：Agent 进程退出时，后台命令的结果不再有意义。如果使用非守护线程，用户按 Ctrl+C 后程序可能不会立即退出（需要等待所有后台命令完成）。

### Q2：为什么 `_notification_queue` 中的结果截断到 500 字符？

```python
# 源码第 87 行
"result": (output or "(no output)")[:500]
```

**原因**：通知会被注入到 LLM 的上下文中。pytest 的完整输出可能超过 50000 字符，直接注入会浪费大量 token。500 字符足以展示测试摘要（如通过的用例数和失败的详情）。

如果需要完整输出，Agent 可以调用 `check_background(task_id=...)` 查看 `self.tasks` 中保存的完整结果（截断到 50000 字符）。

### Q3：`self.tasks` 为什么不需要锁保护？

`self.tasks` 字典的访问没有被 `self._lock` 保护。这在当前实现中是安全的，因为：

1. **`run()` 方法**（主线程）：在后台线程启动前写入 `self.tasks[task_id]`
2. **`_execute()` 方法**（后台线程）：在线程启动后才修改 `self.tasks[task_id]`
3. **`check()` 方法**（主线程）：只读取

由于 Python 的 GIL 和 `dict` 操作的原子性，以及时序上的"先写后改"保证，当前实现不会出现数据竞争。

但在更复杂的场景（如多个后台线程修改同一个 task）中，应该使用锁保护。

### Q4：如何限制并发任务数量？

当前源码没有限制并发数量。如果用户启动 100 个后台任务，会创建 100 个线程。

实现并发限制的方法：

```python
import threading

class BackgroundManager:
    def __init__(self, max_concurrent: int = 5):
        self._semaphore = threading.Semaphore(max_concurrent)

    def run(self, command: str) -> str:
        if not self._semaphore.acquire(blocking=False):
            return "Error: Too many concurrent tasks"
        # ... 启动线程 ...
        # 在 _execute 末尾调用 self._semaphore.release()
```

### Q5：后台任务可以访问文件吗？

可以。后台任务通过 `subprocess.run()` 启动子进程，子进程的工作目录与主线程相同（`cwd=WORKDIR`）。它拥有完全的文件系统访问权限，可以读写主线程操作的相同文件。

**潜在风险**：如果主线程和后台线程同时写入同一个文件，可能导致数据损坏。需要在应用层自行协调。

### Q6：drain_notifications() 返回空但任务已完成，结果去哪了？

可能出现这种"结果丢失"假象的原因和排查方法：

**原因 1：通知已经被上一轮 drain 取走了**

`drain_notifications()` 采用"取出并清空"模式。如果上一轮 Agent 循环中已经 drain 过，结果就已经注入到了上一轮的上下文中。检查 `messages` 列表中是否有包含 `<background-results>` 的 user 消息。

**原因 2：时序窗口问题**

后台线程的 `_execute()` 方法分为两步：更新 `self.tasks` 和 append 到 `_notification_queue`。如果在更新 `self.tasks` 之后、append 队列之前恰好被 drain，任务状态显示 `completed` 但通知尚未入队。

排查方法：调用 `check_background(task_id="...")` 查看任务的完整状态和结果。如果任务确实已完成，结果保存在 `self.tasks` 中，可以正常获取。

**原因 3：守护线程被提前终止**

如果主线程在后台线程完成之前退出（如用户按 Ctrl+C），守护线程会被强制终止，`_execute()` 中的 append 操作不会执行。结果永远丢失。

预防：对于关键任务，不使用 `daemon=True`，或在外部实现结果持久化（如写入临时文件）。

---

## 8. 最佳实践

### 8.1 何时使用后台任务

```
适合后台运行（background_run）：
  - 安装依赖：pip install, npm install（通常 1-5 分钟）
  - 运行测试：pytest, jest（通常 10 秒 - 5 分钟）
  - 构建项目：make, webpack, cargo build（通常 30 秒 - 10 分钟）
  - 数据处理：ETL、数据转换（可能很长）
  - 代码生成：编译器、代码生成工具

不适合后台运行（用同步 bash）：
  - 需要立即结果的命令：git status, ls, cat
  - 相互依赖的命令：命令 B 需要命令 A 的输出
  - 快速命令（< 1 秒）：echo, pwd, date
  - 交互式命令：需要用户输入的命令
```

### 8.2 超时设置原则

| 超时值 | 适用场景 | 源码位置 |
|--------|----------|----------|
| 120 秒 | 同步 bash（s02/s07） | `run_bash()` |
| 300 秒 | 后台任务（s08） | `_execute()` |

后台任务使用更长的超时（300 秒）是因为：
1. 不阻塞主循环，等待时间长一些可以接受
2. 适合的操作（构建、安装、测试）本身就可能需要几分钟
3. 300 秒 = 5 分钟是一个合理的上限，避免失控进程

---

## 9. 架构总结

### 9.1 核心要点

| 要点 | 说明 |
|------|------|
| **守护线程** | `daemon=True` 确保主程序退出时后台线程自动终止 |
| **通知队列** | `threading.Lock` 保护的 `_notification_queue`，后台线程写入、主线程读取 |
| **drain 模式** | 每次 LLM 调用前 `drain_notifications()`，取出并清空队列 |
| **超时保护** | `timeout=300` 秒，防止后台子进程无限运行 |
| **输出截断** | 通知 500 字符（节省 token），完整结果 50000 字符 |
| **渐进增强** | 只在 LLM 调用前增加 drain 逻辑，不修改循环核心结构 |

### 9.2 代码模板

```python
# 启动后台任务
background_run(command="pytest tests/")
# → "Background task a1b2c3d4 started: pytest tests/"

# 查看单个任务状态
check_background(task_id="a1b2c3d4")
# → "[completed] pytest tests/\n15 passed in 2.34s"

# 列出所有后台任务
check_background()
# → "a1b2c3d4: [completed] pytest tests/"

# Agent 循环中自动处理通知（源码自动完成，无需手动调用）
notifs = BG.drain_notifications()
# → [{"task_id": "a1b2c3d4", "status": "completed", "result": "15 passed..."}]
```

### 9.3 关键洞察

```
后台任务的核心设计：

1. 解耦执行时间
   后台线程运行慢命令，主线程继续思考和响应。
   Agent 的"思考速度"不再受限于"命令执行速度"。

2. Drain 模式的精妙之处
   不用回调、不用事件驱动，而是"下次调用前统一检查"。
   这与 Agent 循环的 pull 模式天然契合——
   Agent 主动拉取结果，而不是被推送。

3. 线程安全的最小化
   只对真正需要的共享状态（_notification_queue）加锁。
   self.tasks 利用时序保证而非锁来保证安全，减少锁竞争。

4. 超时保护是必需品
   没有 timeout 参数，一个失控的子进程会永久占用线程。
   300 秒是"足够长但不会无限"的实用上限。
```

---

## 10. 练习

### 练习 1：并发限制

使用 `threading.Semaphore` 限制同时运行的后台任务数量（如最多 5 个）。超出限制时返回错误提示。

**提示**：在 `run()` 中 `acquire()`，在 `_execute()` 末尾 `release()`。

### 练习 2：任务取消

添加 `cancel_background` 工具，使用 `subprocess.Popen` 替代 `subprocess.run`，保留 `Popen` 对象以便调用 `terminate()`。

**提示**：将 `Popen` 对象存入 `self.tasks[task_id]`，取消时调用 `proc.terminate()`。

### 练习 3：进度跟踪

对于支持进度输出的命令（如 `pip install`），实现流式读取 stdout 并定期更新任务状态。

**提示**：使用 `subprocess.Popen` + `proc.stdout.readline()` 循环读取。

### 练习 4：与 s07 集成

实现一个组合场景：当 s07 的某个任务被标记为 `in_progress` 时，自动使用 `background_run` 启动对应的命令。命令完成后自动将任务标记为 `completed`。

**提示**：在 `task_update` 的 `in_progress` 分支中启动后台任务，在通知 drain 回调中检查是否有关联的 s07 任务。

---

## 11. 下一步

现在 Agent 可以在后台运行任务了。但所有工作仍然由单个 Agent 完成，无法利用多个 Agent 的协作能力。

在 [s09：Agent 团队](./s09-agent-teams.md) 中，我们将学习：
- 如何创建多个持久化的 Agent，各自拥有独立的消息列表
- Agent 之间的通信机制（JSONL 邮箱 + MessageBus）
- 团队生命周期管理（创建、运行、销毁）

---

< 上一章：[s07-任务系统](./s07-task-system.md) | [目录](./index.md) | 下一章：[s09-Agent 团队](./s09-agent-teams.md) >
