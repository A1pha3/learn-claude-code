# s08：后台任务 —— 并发执行的艺术

> **核心口号**：*"慢操作后台运行，Agent 继续思考"*

> **学习目标**：理解并发执行的需求，掌握守护线程机制，学会设计通知队列

---

## 学习目标

完成本章后，你将能够：

1. **理解阻塞问题** —— 为什么长时间操作会拖慢 Agent
2. **掌握守护线程** —— 在后台运行子进程
3. **实现通知队列** —— 线程安全的结果收集
4. **设计超时机制** —— 防止任务无限运行

---

## 1. 问题的本质

### 1.1 阻塞式执行的问题

想象一个典型的开发场景：

```
用户：安装依赖并创建配置文件

Agent（阻塞式）：
  1. 运行 pip install（需要 2 分钟）
     Agent 在等待，什么都不做
     ↓
  2. 创建配置文件（5 秒）
     ↓
  3. 返回结果

总时间：2 分 5 秒
```

**问题**：Agent 在等待 `pip install` 时被完全阻塞，无法做其他事情。

### 1.2 理想的方式

```
用户：安装依赖并创建配置文件

Agent（非阻塞）：
  1. 启动 pip install（后台）
     → task_id: bg_123 开始运行
     ↓
  2. 立即创建配置文件（5 秒）
     ↓
  3. 继续其他工作...
     ↓
  4. 后台任务完成
     → 收到通知：bg_123 完成
     ↓
  5. 将结果注入到上下文

总时间：最多 2 分 5 秒
实际并行：pip 与配置文件创建同时进行
```

---

## 2. 后台执行架构

### 2.1 架构图

```
主线程                        后台线程 1              后台线程 2
┌─────────────┐              ┌──────────┐           ┌──────────┐
│  Agent 循环  │              │pip install│           │pytest    │
│             │              │          │           │          │
│  [LLM 调用]  │              │  运行中  │           │  运行中  │
│      ↓      │              │    ↓     │           │    ↓     │
│  drain_queue │◄────────────┤  完成    │           │  运行中  │
│      ↓      │   通知       │    ↓     │           │    ↓     │
│  注入结果   │              │ enqueue  │           │  完成    │
│      ↓      │              │    结果   │           │    ↓     │
│  [LLM 调用]  │◄────────────┤          │◄──────────┤ enqueue  │
│  (带新上下文) │   通知       │          │   通知    │    结果   │
└─────────────┘              └──────────┘           └──────────┘
         ↑                                                   │
         └───────────────── 通知队列 ────────────────────────┘
                    (thread-safe list)
```

### 2.2 时间线对比

```
阻塞式：
├─ pip install (120s) ─┤ ├─ 创建配置 (5s) ─┤
──────────────────────────────────────────────→ 125s

非阻塞：
├─ pip install (120s) ─────────┤
├─ 创建配置 (5s) ─┤ ├─ 其他工作 (10s) ─┤
──────────────────────────────────────────────→ 120s (并行)
```

---

## 3. BackgroundManager 实现

### 3.1 核心类

```python
import threading
import subprocess
import uuid
import time
from typing import Dict, List
from collections import deque

class BackgroundManager:
    def __init__(self):
        self.tasks: Dict[str, Dict] = {}      # task_id -> task_info
        self._notification_queue: List[Dict] = []  # 线程安全的通知队列
        self._lock = threading.Lock()         # 队列访问锁

    def run(self, command: str, timeout: int = 300) -> str:
        """在后台运行命令"""
        task_id = str(uuid.uuid4())[:8]

        # 记录任务
        self.tasks[task_id] = {
            "status": "running",
            "command": command,
            "start_time": time.time()
        }

        # 启动守护线程
        thread = threading.Thread(
            target=self._execute,
            args=(task_id, command, timeout),
            daemon=True  # 守护线程：主程序退出时自动结束
        )
        thread.start()

        return f"后台任务 {task_id} 已启动：{command[:50]}"

    def _execute(self, task_id: str, command: str, timeout: int):
        """线程执行函数"""
        try:
            result = subprocess.run(
                command,
                shell=True,
                cwd=WORKDIR,
                capture_output=True,
                text=True,
                timeout=timeout
            )
            output = (result.stdout + result.stderr).strip()[:5000]
            status = "completed"
        except subprocess.TimeoutExpired:
            output = f"超时（{timeout}秒）"
            status = "timeout"
        except Exception as e:
            output = f"错误：{str(e)}"
            status = "failed"

        # 更新任务状态
        with self._lock:
            self.tasks[task_id]["status"] = status
            self.tasks[task_id]["output"] = output
            self.tasks[task_id]["end_time"] = time.time()

            # 将结果加入通知队列
            self._notification_queue.append({
                "task_id": task_id,
                "status": status,
                "output": output[:500]  # 限制通知大小
            })

    def check(self, task_id: str) -> str:
        """检查任务状态"""
        task = self.tasks.get(task_id)
        if not task:
            return f"任务不存在：{task_id}"

        status = task["status"]
        if status == "running":
            elapsed = time.time() - task["start_time"]
            return f"任务 {task_id} 运行中（已 {elapsed:.0f} 秒）"
        else:
            return f"任务 {task_id} 状态：{status}\n输出：{task.get('output', '')}"

    def drain_notifications(self) -> List[Dict]:
        """取出所有通知（清空队列）"""
        with self._lock:
            notifications = self._notification_queue.copy()
            self._notification_queue.clear()
        return notifications

    def list_all(self) -> str:
        """列出所有后台任务"""
        if not self.tasks:
            return "无后台任务"

        lines = ["后台任务列表："]
        for task_id, task in sorted(self.tasks.items()):
            status = task["status"]
            command = task["command"][:40]
            lines.append(f"  [{status}] {task_id}: {command}")

        return "\n".join(lines)
```

### 3.2 线程安全的关键

```python
# 为什么需要锁？
_notification_queue: List[Dict] = []
_lock = threading.Lock()

# 写入队列（后台线程）
with self._lock:
    self._notification_queue.append(notification)

# 读取队列（主线程）
with self._lock:
    notifications = self._notification_queue.copy()
    self._notification_queue.clear()
```

**没有锁的问题**：两个线程同时修改列表可能导致数据损坏。

---

## 4. 集成到 Agent 循环

### 4.1 工具定义

```python
TOOLS = [
    # ... 其他工具 ...
    {
        "name": "background_run",
        "description": "在后台运行命令，立即返回。适用于耗时操作如安装依赖、运行测试。",
        "input_schema": {
            "type": "object",
            "properties": {
                "command": {
                    "type": "string",
                    "description": "要执行的命令"
                },
                "timeout": {
                    "type": "integer",
                    "description": "超时时间（秒），默认 300",
                    "default": 300
                }
            },
            "required": ["command"]
        }
    },
    {
        "name": "background_check",
        "description": "检查后台任务的状态",
        "input_schema": {
            "type": "object",
            "properties": {
                "task_id": {
                    "type": "string",
                    "description": "任务 ID"
                }
            },
            "required": ["task_id"]
        }
    },
    {
        "name": "background_list",
        "description": "列出所有后台任务",
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
BG = BackgroundManager()

TOOL_HANDLERS = {
    # ... 其他工具 ...
    "background_run": lambda **kw: BG.run(kw["command"], kw.get("timeout", 300)),
    "background_check": lambda **kw: BG.check(kw["task_id"]),
    "background_list": lambda **kw: BG.list_all(),
}
```

### 4.3 在循环中处理通知

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]

    while True:
        # ========== 处理后台通知 ==========
        notifications = BG.drain_notifications()
        if notifications:
            # 格式化通知
            notif_texts = []
            for n in notifications:
                notif_texts.append(
                    f"[后台任务 {n['task_id']}] 状态：{n['status']}\n{n['output']}"
                )

            # 注入到消息
            messages.append({
                "role": "user",
                "content": f"<background-results>\n" + "\n".join(notif_texts) + "\n</background-results>"
            })
            messages.append({
                "role": "assistant",
                "content": "收到后台任务结果。"
            })

        # 正常的 LLM 调用
        response = client.messages.create(
            model=MODEL,
            messages=messages,
            tools=TOOLS,
            max_tokens=8000,
        )
        # ... 其余逻辑
```

---

## 5. 完整示例

### 5.1 并发执行多个任务

```
用户：运行测试并创建新的测试文件

Agent：
  1. background_run(command="pytest tests/")
     → 后台任务 abc123 已启动

  2. 立即创建测试文件
     write_file("tests/test_new.py", "...")

  3. 继续其他工作...

  ...（pytest 在后台运行）...

  4. pytest 完成！
     <background-results>
     [后台任务 abc123] 状态：completed
     ==================== test session starts ====================
     collected 15 items
     tests/test_auth.py ....                                   [ 26%]
     tests/test_db.py ......                                    [ 66%]
     tests/test_api.py .....                                    [100%]
     ==================== 15 passed in 2.34s =====================
     </background-results>

  5. Agent 处理测试结果并继续
```

### 5.2 长时间运行的任务

```
用户：安装项目依赖

Agent：
  background_run(command="pip install -r requirements.txt")
  → 后台任务 def456 已启动

用户（立即）：同时创建 README
Agent：创建 README.md...

用户：检查安装状态
Agent：background_check(task_id="def456")
  → 任务 def456 运行中（已 45 秒）

用户（稍后）：再检查一次
Agent：background_check(task_id="def456")
  → 任务 def456 状态：completed
  → 输出：Successfully installed anthropic-0.18.0...
```

---

## 6. 超时和错误处理

### 6.1 超时机制

```python
def _execute(self, task_id: str, command: str, timeout: int):
    try:
        result = subprocess.run(
            command,
            shell=True,
            timeout=timeout  # 超时限制
        )
    except subprocess.TimeoutExpired:
        # 超时处理
        with self._lock:
            self.tasks[task_id]["status"] = "timeout"
            self.tasks[task_id]["output"] = f"命令超时（{timeout}秒）"
            self._notification_queue.append({
                "task_id": task_id,
                "status": "timeout",
                "output": f"超时：{command[:50]}"
            })
```

### 6.2 错误处理

```python
# 命令执行失败
result = subprocess.run(...)
if result.returncode != 0:
    # 非零退出码表示错误
    error_msg = result.stderr.strip()
    # 仍然视为"完成"，但输出包含错误信息
```

### 6.3 任务清理

```python
def cleanup_old_tasks(self, max_age: int = 3600):
    """清理旧任务（超过 max_age 秒）"""
    now = time.time()
    to_remove = []

    for task_id, task in self.tasks.items():
        if task["status"] in ["completed", "failed", "timeout"]:
            age = now - task.get("end_time", 0)
            if age > max_age:
                to_remove.append(task_id)

    for task_id in to_remove:
        del self.tasks[task_id]

    return f"清理了 {len(to_remove)} 个旧任务"
```

---

## 7. 与 s07 的对比

| 方面 | s07 任务系统 | s08 后台任务 |
|------|-------------|--------------|
| 执行方式 | 同步 | 异步 |
| 存储 | 文件系统 | 内存 |
| 状态管理 | pending/in_progress/completed | running/completed/failed/timeout |
| 适用场景 | 逻辑任务 | 耗时命令 |
| 通知机制 | 无 | 通知队列 |

---

## 8. 常见问题

### Q1：为什么使用守护线程？

```python
thread = threading.Thread(..., daemon=True)
```

**答案**：守护线程在主程序退出时自动结束，防止程序挂起。

非守护线程会阻止程序退出，即使主线程已经完成。

### Q2：后台任务可以访问文件吗？

**答案**：可以。后台任务是在同一工作目录中运行的子进程，可以访问相同的文件系统。

### Q3：如何限制并发任务数量？

```python
class BackgroundManager:
    def __init__(self, max_concurrent: int = 5):
        self.max_concurrent = max_concurrent
        self._semaphore = threading.Semaphore(max_concurrent)

    def run(self, command: str, timeout: int = 300) -> str:
        # 等待可用槽位
        self._semaphore.acquire()
        try:
            # 启动线程...
            thread.start()
        finally:
            # 线程完成后释放
            self._semaphore.release()
```

### Q4：后台任务的输出太大怎么办？

```python
# 限制输出大小
output = (result.stdout + result.stderr).strip()[:5000]

# 或者写入文件
output_file = TASKS_DIR / f"{task_id}.output"
output_file.write_text(result.stdout + result.stderr)
```

---

## 9. 最佳实践

### 9.1 何时使用后台任务

```
✅ 适合后台运行：
- 安装依赖（pip install, npm install）
- 运行测试（pytest, jest）
- 构建项目（make, webpack）
- 数据处理（ETL 任务）
- 代码生成（编译器）

❌ 不适合后台运行：
- 需要立即结果的命令
- 相互依赖的命令
- 快速命令（< 1 秒）
```

### 9.2 超时设置

```python
# 根据命令类型设置超时
TIMEOUTS = {
    "pip install": 600,   # 10 分钟
    "npm install": 600,
    "pytest": 300,        # 5 分钟
    "make": 600,
    "default": 60
}

def get_timeout(command: str) -> int:
    for prefix, timeout in TIMEOUTS.items():
        if command.startswith(prefix):
            return timeout
    return TIMEOUTS["default"]
```

### 9.3 任务命名

```python
# 添加可读的任务名称
self.tasks[task_id] = {
    "status": "running",
    "command": command,
    "name": f"{command.split()[0]} @ {time.strftime('%H:%M')}",  # "pytest @ 14:30"
    "start_time": time.time()
}
```

---

## 10. 小结

### 10.1 核心要点

| 要点 | 说明 |
|------|------|
| **守护线程** | 在后台运行子进程 |
| **通知队列** | 线程安全的结果收集 |
| **超时机制** | 防止任务无限运行 |
| **并发执行** | Agent 可以继续思考 |

### 10.2 代码模板

```python
# 启动后台任务
background_run(command="pytest tests/", timeout=300)

# 检查状态
background_check(task_id="abc123")

# 列出所有任务
background_list()

# Agent 循环中自动处理通知
notifications = BG.drain_notifications()
if notifications:
    # 注入到上下文
```

### 10.3 关键洞察

```
后台任务的核心价值：
1. 解耦执行时间：不阻塞 Agent 思考
2. 并发执行：多个任务同时运行
3. 通知机制：完成时自动通知
4. 状态跟踪：随时查询进度

设计原则：
- 守护线程：主程序退出时自动结束
- 线程安全：用锁保护共享状态
- 超时保护：防止任务无限运行
- 限制输出：避免占用过多内存
```

---

## 11. 练习

### 练习 1：任务优先级

实现优先级队列，高优先级的通知优先处理。

### 练习 2：任务取消

添加 `background_cancel` 工具，终止运行中的后台任务。

### 练习 3：进度跟踪

对于长时间任务，定期输出进度信息。

### 练习 4：任务组

允许将多个任务分组，一起管理（如 `group("tests", [pytest, coverage])`）。

---

## 12. 下一步

现在 Agent 可以在后台运行任务了。但所有工作仍然由单个 Agent 完成，无法利用多个 Agent 的协作能力。

在 [s09：Agent 团队](./s09-agent-teams.md) 中，我们将学习：
- 如何创建多个持久化的 Agent
- Agent 之间的通信机制
- 团队生命周期管理

---

< 上一章：[s07-任务系统](./s07-task-system.md) | [目录](./index.md) | 下一章：[s09-Agent 团队](./s09-agent-teams.md) >
