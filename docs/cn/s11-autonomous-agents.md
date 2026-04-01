# s11：自主 Agent —— 自我驱动的团队

> **核心口号**：*"Agent finds work itself."*

> **学习目标**：理解自主性设计，掌握空闲轮询机制，学会任务自动认领与身份重注入

---

## 学习目标

完成本章后，你将能够：

1. **理解自主性的价值** —— 从被动执行到主动认领
2. **掌握 WORK/IDLE 循环** —— WORK 最多 50 轮、IDLE 每 5 秒轮询、60 秒超时
3. **理解任务扫描与认领** —— `scan_unclaimed_tasks` 的三重过滤条件与 `_claim_lock` 竞态防护
4. **理解身份重注入** —— 压缩后在 messages 列表头部插入 identity block
5. **掌握模块级请求跟踪器** —— `shutdown_requests` / `plan_requests` 字典 + `_tracker_lock`

**学习分层表**：

| 层级 | 目标 | 检验标准 |
|------|------|----------|
| ⭐ 基础 | 理解自主性设计的动机 | 能解释 WORK/IDLE 循环的三个阶段及触发条件，理解 60 秒超时的意义 |
| ⭐⭐ 进阶 | 独立配置自主团队完成实际任务 | 能设计合理的任务板（`scan_unclaimed_tasks` 三重过滤），理解 `_claim_lock` 竞态防护 |
| ⭐⭐⭐ 专家 | 优化自主行为策略与身份管理 | 能实现角色偏好过滤、动态超时、认领冲突检测，能评估身份重注入的启发式阈值 |

---

## 0. 上手演练（建议先做）

先运行自主 Agent 示例，再阅读 WORK/IDLE 循环：

```bash
uv run python agents/s11_autonomous_agents.py
```

建议观察：

1. 队友何时从 WORK 切换到 IDLE（LLM 停止调用工具或主动调用 `idle` 工具）。
2. 空闲状态下如何每 5 秒扫描一次任务板，并自动认领无主任务。
3. 60 秒内没有新消息和可认领任务时，队友如何自动 shutdown。
4. 用 `/tasks` 命令观察任务状态从 `[ ]` 变为 `[>]` 的过程。

---

## 1. 从被动到主动

### 1.1 回顾 s09-s10：被动执行

```
Lead: spawn(name="alice", role="coder")
Lead: send(to="alice", content="请实现用户登录")
Lead: send(to="bob", content="请编写测试")
Lead: send(to="alice", content="请修复 bug")
Lead: send(to="bob", content="请审查代码")
...

问题：Lead 需要手动分配每个任务，队友完全被动等待指令
```

### 1.2 自主执行

```
Lead: spawn(name="alice", role="coder")
Lead: spawn(name="bob", role="tester")

[Alice 的空闲循环]
  扫描 .tasks/ 目录 → 发现 task_1.json (pending, 无 owner)
  claim_task(id=1, owner="alice") → 状态变为 in_progress
  执行任务 → 完成

[Bob 的空闲循环]
  扫描 .tasks/ 目录 → 发现 task_2.json (pending, 无 owner)
  claim_task(id=2, owner="bob") → 状态变为 in_progress
  执行任务 → 完成

[完成后的 Alice]
  再次扫描 → 发现 task_3.json (pending, 无 owner)
  自动认领并执行
...
```

**核心变化**：Lead 只负责创建任务和 spawn 队友，队友自己找到工作。

---

## 2. WORK/IDLE 循环

### 2.1 状态机

```
         spawn()
           │
           ▼
    ┌──────────────┐
    │    WORK      │ ◄────── 收到消息 / 认领任务
    │  (执行任务)   │
    └──────┬───────┘
           │
           │ stop_reason != "tool_use"
           │ 或调用 idle 工具
           ▼
    ┌──────────────┐
    │    IDLE      │    每 5 秒轮询一次
    │  (等待工作)   │    最多等待 60 秒
    └──────┬───────┘
           │
     ┌─────┴──────────┐
     │                 │
  收到消息          60 秒超时
  或发现任务        无新工作
     │                 │
     ▼                 ▼
   WORK             SHUTDOWN
```

**三状态值**（对应源码 `_set_status` 中的取值）：
- `working` —— 队友正在执行任务
- `idle` —— 队友在轮询等待新工作
- `shutdown` —— 队友已退出（60 秒超时或收到 shutdown_request）

### 2.2 源码中的循环实现

以下代码直接对应 `TeammateManager._loop` 方法（源码第 207-292 行）：

```python
def _loop(self, name: str, role: str, prompt: str):
    team_name = self.config["team_name"]
    sys_prompt = (
        f"You are '{name}', role: {role}, team: {team_name}, at {WORKDIR}. "
        f"Use idle tool when you have no more work. You will auto-claim new tasks."
    )
    messages = [{"role": "user", "content": prompt}]
    tools = self._teammate_tools()

    while True:
        # ========== WORK 阶段：最多 50 轮 ==========
        for _ in range(50):
            inbox = BUS.read_inbox(name)       # 每轮开头先读收件箱
            for msg in inbox:
                if msg.get("type") == "shutdown_request":
                    self._set_status(name, "shutdown")
                    return                     # 收到关机请求，直接退出
                messages.append({"role": "user", "content": json.dumps(msg)})

            response = client.messages.create(
                model=MODEL, system=sys_prompt,
                messages=messages, tools=tools, max_tokens=8000,
            )
            messages.append({"role": "assistant", "content": response.content})

            if response.stop_reason != "tool_use":
                break  # LLM 停止调用工具，进入 IDLE

            results = []
            idle_requested = False
            for block in response.content:
                if block.type == "tool_use":
                    if block.name == "idle":
                        idle_requested = True   # LLM 主动请求进入空闲
                        output = "Entering idle phase. Will poll for new tasks."
                    else:
                        output = self._exec(name, block.name, block.input)
                    results.append({"type": "tool_result", ...})
            messages.append({"role": "user", "content": results})
            if idle_requested:
                break  # 主动进入 IDLE

        # ========== IDLE 阶段：每 5 秒轮询，最多 60 秒 ==========
        self._set_status(name, "idle")
        resume = False
        polls = IDLE_TIMEOUT // max(POLL_INTERVAL, 1)  # 60 // 5 = 12 次

        for _ in range(polls):
            time.sleep(POLL_INTERVAL)          # 等待 5 秒

            # 优先级 1：检查收件箱
            inbox = BUS.read_inbox(name)
            if inbox:
                for msg in inbox:
                    if msg.get("type") == "shutdown_request":
                        self._set_status(name, "shutdown")
                        return
                    messages.append({"role": "user", "content": json.dumps(msg)})
                resume = True
                break

            # 优先级 2：扫描任务板
            unclaimed = scan_unclaimed_tasks()
            if unclaimed:
                task = unclaimed[0]
                claim_task(task["id"], name)
                # ... 注入任务消息到 messages ...
                resume = True
                break

        if not resume:
            self._set_status(name, "shutdown")
            return    # 60 秒无工作，退出

        self._set_status(name, "working")
        # 回到 WORK 阶段，继续 while True
```

**关键数值**：
| 参数 | 值 | 含义 |
|------|-----|------|
| `POLL_INTERVAL` | 5 | IDLE 阶段每 5 秒轮询一次 |
| `IDLE_TIMEOUT` | 60 | IDLE 阶段总超时 60 秒 |
| WORK 阶段轮数上限 | 50 | `_loop` 中 `for _ in range(50)` |
| IDLE 轮询次数 | 12 | `60 // 5 = 12` |

---

## 3. 空闲轮询

### 3.1 轮询逻辑详解

IDLE 阶段的核心是**优先级分明的两级检查**：

```
每 5 秒一次轮询：
  1. 检查 inbox → 有消息？→ 恢复 WORK
  2. 扫描 .tasks/ → 有未认领任务？→ 认领并恢复 WORK
  3. 都没有 → 继续等待
  ...
  12 次都无结果 → SHUTDOWN
```

**收件箱优先级高于任务扫描**：如果同时有消息和未认领任务，先处理消息。

### 3.2 任务扫描

`scan_unclaimed_tasks()` 是一个**模块级函数**（非类方法），对应源码第 126-135 行：

```python
def scan_unclaimed_tasks() -> list:
    """扫描 .tasks/ 目录中所有 task_*.json，返回符合条件的任务"""
    TASKS_DIR.mkdir(exist_ok=True)
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        # 三重过滤条件
        if (task.get("status") == "pending"      # 1. 状态为 pending
                and not task.get("owner")         # 2. 没有 owner
                and not task.get("blockedBy")):   # 3. 没有阻塞依赖
            unclaimed.append(task)
    return unclaimed
```

**三重过滤的含义**：
1. `status == "pending"` —— 只看未开始的任务
2. `not owner` —— 排除已被认领的任务
3. `not blockedBy` —— 排除被其他任务阻塞的任务（依赖管理）

### 3.3 任务认领与竞态防护

`claim_task()` 使用 `_claim_lock`（`threading.Lock()`）防止并发认领冲突：

```python
_claim_lock = threading.Lock()  # 模块级锁（源码第 76 行）

def claim_task(task_id: int, owner: str) -> str:
    with _claim_lock:                         # 整个读写过程在锁内
        path = TASKS_DIR / f"task_{task_id}.json"
        if not path.exists():
            return f"Error: Task {task_id} not found"
        task = json.loads(path.read_text())
        task["owner"] = owner                 # 设置所有者
        task["status"] = "in_progress"        # 更新状态
        path.write_text(json.dumps(task, indent=2))
    return f"Claimed task #{task_id} for {owner}"
```

**为什么需要锁？** 多个队友线程可能同时扫描到同一个任务。如果没有锁，两个线程可能同时写入 `task_1.json`，导致 owner 被覆盖。`_claim_lock` 保证同一时刻只有一个线程能执行认领的读-改-写操作。

**注意**：锁防止并发写入，但不防止已认领任务的覆盖——因为 `claim_task` 没有在锁内检查任务是否已被认领。如果线程 A 在锁外扫描到任务、线程 B 也扫描到同一任务，B 先获得锁并认领成功，A 随后获得锁时会直接覆盖 B 的 owner 而不做检查。这意味着"先到先得"实际上由线程调度决定，后进入锁的线程会无条件覆盖前者的认领结果。

---

## 4. 身份重注入

### 4.1 问题

```
Alice 的上下文：
[系统提示：You are 'alice', role: coder, team: my-team]
[用户：实现用户登录]
[工具调用：write_file]
[工具结果：...]
...  （几十轮对话）
[上下文压缩发生！]
[压缩摘要：Alice 完成了登录功能]

压缩后 messages 列表可能变得很短（<= 3 条），
此时系统提示中的身份信息虽然存在，但 messages 上下文中
缺少足够的对话历史来维持身份感知。
```

### 4.2 实际实现

身份重注入发生在 IDLE 阶段扫描到未认领任务时（源码第 281-285 行）：

```python
# 在 IDLE 阶段认领任务时，检查是否需要重注入
if len(messages) <= 3:
    messages.insert(0, make_identity_block(name, role, team_name))
    messages.insert(1, {"role": "assistant", "content": f"I am {name}. Continuing."})

# 然后追加任务消息
messages.append({"role": "user", "content": task_prompt})
messages.append({"role": "assistant", "content": f"Claimed task #{task['id']}. Working on it."})
```

`make_identity_block` 函数（源码第 151-155 行）：

```python
def make_identity_block(name: str, role: str, team_name: str) -> dict:
    return {
        "role": "user",
        "content": (
            f"<identity>You are '{name}', role: {role}, team: {team_name}. "
            f"Continue your work.</identity>"
        ),
    }
```

**重注入的位置**：`messages.insert(0, ...)` 将 identity block 插入到 messages 列表的最前面（头部），紧接着插入一条 assistant 确认消息。

**触发条件**：`len(messages) <= 3` —— 当对话历史很短（可能刚被压缩）时才触发。这是一种启发式判断：正常工作中的 messages 列表通常很长，如果突然变短，说明可能发生了上下文压缩。

---

## 5. 完整流程

### 5.1 初始化

```
s11 >>
Lead 通过 agent_loop 与 LLM 交互，
使用 spawn_teammate 工具创建队友。

LLM 调用：spawn_teammate(name="alice", role="coder", prompt="负责后端 API 开发")
LLM 调用：spawn_teammate(name="bob", role="tester", prompt="负责测试和质量保证")

Lead 使用 write_file 创建任务文件：
  .tasks/task_1.json: {"id": 1, "subject": "实现用户 API", "status": "pending"}
  .tasks/task_2.json: {"id": 2, "subject": "实现认证 API", "status": "pending"}
  .tasks/task_3.json: {"id": 3, "subject": "编写 API 测试", "status": "pending"}
```

### 5.2 自主认领

```
[Alice 的线程 - WORK 阶段]
  启动，进入 WORK
  LLM：我需要查看任务板
        bash("ls .tasks/")
  结果：task_1.json  task_2.json  task_3.json

  LLM：我认领任务 #1
        claim_task(task_id=1)
  [claim_task 执行：owner="alice", status="in_progress"]

  LLM：开始实现用户 API
        write_file("api/users.py", ...)
        [工作...]

  LLM：任务完成，没有更多工作
        idle()
  [进入 IDLE 阶段]

[Alice 的线程 - IDLE 阶段]
  状态变为 idle
  第 1 次轮询（5 秒后）：收件箱空，扫描任务板 → 发现 task_2.json 待认领
  claim_task(id=2, owner="alice")
  [插入 identity block（如果 messages <= 3）]
  [追加任务消息到 messages]
  状态变为 working
  返回 WORK 阶段

[Alice 的线程 - WORK 阶段]
  LLM 看到新认领的任务 #2，开始实现认证 API
  ...
```

### 5.3 并行执行

```
时间线：
[Alice]                         [Bob]
├── WORK: 认领并执行 task #1    ├── WORK: 认领并执行 task #3
├── IDLE: 5s 后发现 task #2     ├── IDLE: 扫描无新任务
├── WORK: 执行 task #2          ├── IDLE: 继续等待...
├── IDLE: 扫描无新任务          ├── IDLE: ...60 秒超时
└── IDLE: ...60 秒超时          └── SHUTDOWN
└── SHUTDOWN

两个队友自主并行工作，Lead 只需创建任务
```

### 5.4 交互命令

在 REPL 循环中，除了正常的对话输入外，还支持三个快捷命令：

| 命令 | 功能 |
|------|------|
| `/team` | 查看所有队友及其状态 |
| `/inbox` | 查看 lead 的收件箱 |
| `/tasks` | 查看 `.tasks/` 中所有任务的状态 |

`/tasks` 的输出格式：
```
  [ ] #1: 实现用户 API
  [>] #2: 实现认证 API @alice
  [x] #3: 编写 API 测试 @bob
```

---

## 6. 工具集成

### 6.1 Lead 工具（14 个）

Lead 的 `TOOLS` 列表（源码第 477-506 行）包含 14 个工具：

| 工具名 | 用途 |
|--------|------|
| `bash` | 执行 shell 命令 |
| `read_file` | 读取文件 |
| `write_file` | 写入文件 |
| `edit_file` | 替换文件中的文本 |
| `spawn_teammate` | 创建自主队友 |
| `list_teammates` | 列出所有队友 |
| `send_message` | 发消息给队友 |
| `read_inbox` | 读取 lead 收件箱 |
| `broadcast` | 广播消息给所有队友 |
| `shutdown_request` | 请求队友关机 |
| `shutdown_response` | 查看关机请求状态（注意：在 lead 端是查看状态） |
| `plan_approval` | 审批队友的计划 |
| `idle` | Lead 不使用（返回 "Lead does not idle."） |
| `claim_task` | Lead 也可以手动认领任务 |

### 6.2 Teammate 工具（10 个）

每个队友通过 `_teammate_tools()` 获得的工具列表（源码第 332-355 行）：

| 工具名 | 用途 |
|--------|------|
| `bash` | 执行 shell 命令 |
| `read_file` | 读取文件 |
| `write_file` | 写入文件 |
| `edit_file` | 替换文件中的文本 |
| `send_message` | 发消息给其他队友或 lead |
| `read_inbox` | 读取自己的收件箱 |
| `shutdown_response` | 响应关机请求（approve/reject） |
| `plan_approval` | 提交计划等待 lead 审批 |
| `idle` | 主动进入空闲轮询阶段 |
| `claim_task` | 手动认领指定任务 |

### 6.3 消息类型

`VALID_MSG_TYPES`（源码第 64-70 行）定义了所有合法的消息类型：

```python
VALID_MSG_TYPES = {
    "message",                 # 普通消息
    "broadcast",               # 广播消息
    "shutdown_request",        # 关机请求
    "shutdown_response",       # 关机响应
    "plan_approval_response",  # 计划审批响应
}
```

### 6.4 请求跟踪器

两个模块级字典用于跟踪异步请求（源码第 73-74 行）：

```python
shutdown_requests = {}   # {request_id: {"target": name, "status": "pending/approved/rejected"}}
plan_requests = {}       # {request_id: {"from": name, "plan": text, "status": "pending/approved/rejected"}}
```

两个锁保护并发访问：

```python
_tracker_lock = threading.Lock()   # 保护 shutdown_requests 和 plan_requests
_claim_lock = threading.Lock()     # 保护 claim_task 的读-改-写
```

---

## 7. 与 s10 的对比

| 方面 | s10 协议 | s11 自主 |
|------|----------|---------|
| 任务分配 | Lead 手动分配 | 队友自动扫描认领 |
| 触发方式 | 收到消息 | 收到消息 + 定期扫描任务板 |
| 状态值 | working/idle/shutdown | 相同的三状态值 |
| 空闲行为 | 无（队友停了就停了） | WORK/IDLE 循环 + 60 秒超时 |
| 身份维护 | 系统提示固定 | + 压缩后身份重注入 |
| 协议机制 | shutdown/plan 请求-响应 | 继承 s10 的协议机制 |
| 并发保护 | 无 | `_claim_lock` + `_tracker_lock` |
| 架构 | Lead + Teammate 消息总线 | 继承 + 线程化 TeammateManager |

---

## 8. 常见问题

### Q1：多个队友同时认领同一任务怎么办？

源码使用 `_claim_lock`（`threading.Lock()`）来防止竞态条件。`claim_task` 函数的整个读-改-写过程在 `with _claim_lock:` 块内执行，保证同一时刻只有一个线程能修改任务文件。

```python
def claim_task(task_id: int, owner: str) -> str:
    with _claim_lock:                              # 线程锁
        path = TASKS_DIR / f"task_{task_id}.json"
        task = json.loads(path.read_text())
        task["owner"] = owner
        task["status"] = "in_progress"
        path.write_text(json.dumps(task, indent=2))
    return f"Claimed task #{task_id} for {owner}"
```

但要明确：锁防止并发写入，但不防止已认领任务的覆盖——因为 `claim_task` 没有在锁内检查任务是否已被认领（例如检查 `task.get("owner")` 是否非空）。如果线程 A 在锁外扫描到任务、线程 B 也扫描到同一任务，B 先获得锁并认领成功，A 随后获得锁时会直接覆盖 B 的 owner 而不做检查。这是当前实现的简化之处，练习 2 提供了修复思路。

### Q2：IDLE 阶段收件箱和任务板的优先级？

收件箱优先级更高。源码中先检查 `BUS.read_inbox(name)`，如果有消息则立即恢复 WORK；只有收件箱为空时才扫描任务板。这确保紧急消息（如 shutdown_request）能被及时处理。

### Q3：身份重注入为什么只在 messages <= 3 时触发？

这是一种**启发式判断**。正常工作中的 messages 列表通常有几十条消息；如果突然降到 3 条以下，最可能的原因是上下文压缩（context compression）。此时在列表头部插入 identity block，帮助 LLM 恢复对自身角色的认知。

具体来说，选择 `<= 3` 这个阈值是因为：上下文压缩发生后，messages 列表通常只剩 2 条消息——一条是压缩摘要（compression summary），另一条是对压缩的确认回复。加上系统提示本身不计入 messages，因此 `<= 3` 正好覆盖了"刚被压缩"的场景。如果阈值设得太大，会在正常工作中频繁误触发；设得太小则可能漏掉刚压缩后的情况。

### Q4：WORK 阶段最多 50 轮，够用吗？

50 轮对应 `for _ in range(50)` 的循环上限。每轮包含一次 LLM 调用和可能的多步工具执行。对于大多数任务，50 轮绰绰有余。如果用完 50 轮，循环自然退出进入 IDLE 阶段，不会报错。

### 队友异常行为排查

**场景 1：队友认领了不适合的任务**

症状：一个 `tester` 角色的队友认领了 `实现用户 API` 这样的开发任务。

原因分析：`scan_unclaimed_tasks()` 只根据三重过滤条件（pending、无 owner、无 blockedBy）筛选任务，不考虑任务内容与队友角色的匹配度。所有队友看到的是同一个任务列表，先到先得。

排查步骤：
1. 用 `/tasks` 命令查看哪些任务被谁认领，检查 owner 是否与角色匹配。
2. 查看该队友的 system prompt 是否明确描述了其职责范围——如果 prompt 只写了"你是 tester"而没有排除开发类任务，LLM 可能不会主动跳过。
3. 解决方案：在任务的 JSON 中添加 `required_role` 字段，在 `scan_unclaimed_tasks()` 中根据队友 role 进行过滤（参见练习 3 的思路）。

**场景 2：队友在 IDLE 阶段不认领任务**

症状：任务板上有多个 pending 任务，但某个队友一直处于 idle 状态，不主动认领。

原因分析：可能有以下几种情况：
1. 队友的 IDLE 轮询已超时（60 秒）并进入 shutdown 状态——此时该队友线程已退出，不会再扫描任务。用 `/team` 命令查看其状态是否为 shutdown。
2. 队友恰好在上一次轮询间隙收到了一条消息，进入了 WORK 阶段但 LLM 没有选择认领任务——检查该队友的收件箱是否堆积了未处理的消息。
3. 任务被其他队友抢先认领——在并发环境下，多个 idle 队友同时扫描到同一任务时，只有最先获得 `_claim_lock` 的队友能成功认领。

排查步骤：
1. 用 `/team` 查看队友状态，如果是 `shutdown` 则需要重新 spawn。
2. 用 `/tasks` 确认任务确实处于 pending 且无 owner。
3. 检查 `.tasks/` 目录下对应的 `task_*.json` 文件内容，确认 status 和 owner 字段。

---

## 实战应用：构建自主工作团队

### 场景一：看板驱动开发（自动认领 Jira/GitHub Issues）

将外部 Issue 转换为 `.tasks/` 中的任务文件，自主团队自动认领：

```
Lead 工作流：
  1. 从 GitHub Issues 拉取待办事项
  2. 为每个 Issue 创建 task_*.json：
     {
       "id": 1,
       "subject": "Fix: 登录页面在 Safari 下布局错乱",
       "description": "Issue #42: Safari 下 flexbox 兼容性问题",
       "status": "pending",
       "owner": "",
       "required_role": "frontend"    ← 扩展字段，可让 scan 函数做角色过滤
     }
  3. spawn 队友，团队自动运转

Alice (前端) 的 IDLE 轮询：
  → 扫描任务板 → 发现 task_1 (pending, required_role=frontend)
  → claim_task(id=1, owner="alice")
  → WORK: 修复 Safari 布局 → 提交 → idle()
  → IDLE: 继续扫描...
```

### 场景二：运维自愈系统（故障检测 → 诊断 → 修复 → 验证）

```
Lead (运维负责人)
 ├─ Watchdog (监控 Agent)
 │   prompt: "每 30 秒检查服务健康状态，发现异常立即创建诊断任务"
 │   行为：IDLE 轮询中通过 bash 执行健康检查 → 发现故障 → 创建任务
 │
 ├─ Doctor (诊断 Agent)
 │   prompt: "认领诊断任务，分析日志，输出根因报告"
 │   行为：扫描到诊断类任务 → 认领 → 查日志 → 输出根因 → 创建修复任务
 │
 └─ Medic (修复 Agent)
     prompt: "认领修复任务，执行预定义修复方案，修复后触发验证"
     行为：扫描到修复类任务 → 认领 → 执行修复 → 创建验证任务

自愈闭环：
  Watchdog 检测故障 → Doctor 诊断 → Medic 修复 → Watchdog 再次验证
  全程无需人工介入
```

### 场景三：内容生产流水线

```
Lead 创建任务：
  task_1: "撰写 AI Agent 技术趋势报告" (status: pending)
  task_2: "制作配套图表" (status: pending, blockedBy: [1])
  task_3: "排版与校对" (status: pending, blockedBy: [1, 2])

自主团队：
  Writer (IDLE → 扫描 → 认领 task_1 → WORK → 完成 → idle)
  Designer (IDLE → 扫描 → task_2 被 blockedBy 跳过 → 继续等待)
  → task_1 完成，_clear_dependency 解锁 task_2
  Designer (IDLE → 扫描 → 认领 task_2 → WORK → 完成 → idle)
  → task_2 完成，_clear_dependency 解锁 task_3
  Editor (IDLE → 扫描 → 认领 task_3 → WORK → 完成)
```

### "被动 vs 自主"升级决策表

| 评估维度 | 被动模式 (s10) | 自主模式 (s11) |
|----------|----------------|----------------|
| 任务来源 | Lead 逐一分配 | 队友扫描任务板自动认领 |
| Lead 负担 | 高（每步都需要手动调度） | 低（只管创建任务） |
| 适用规模 | 3 个以内队友、5 个以内任务 | 3+ 队友、10+ 任务 |
| 响应速度 | 依赖 Lead 的调度频率 | 5 秒轮询，几乎实时 |
| 资源管理 | 队友持续占用线程 | 60 秒无工作自动释放 |
| 一致性要求 | Lead 控制分配，无竞态 | 需要 `_claim_lock` 防竞态 |
| 错误恢复 | Lead 重新分配即可 | 需要处理认领冲突和僵尸任务 |
| 升级成本 | 低 | 需要设计合理的任务板和角色过滤 |

---

## 9. 小结

### 9.1 核心要点

| 要点 | 说明 |
|------|------|
| **自主性** | 队友通过 IDLE 轮询主动发现并认领任务 |
| **WORK/IDLE 循环** | WORK 最多 50 轮执行，IDLE 每 5 秒轮询、60 秒超时 |
| **任务扫描三重过滤** | pending + 无 owner + 无 blockedBy |
| **竞态防护** | `_claim_lock` 保护认领操作的原子性 |
| **身份重注入** | messages <= 3 时在列表头部插入 identity block |
| **请求跟踪器** | `shutdown_requests` / `plan_requests` + `_tracker_lock` |

### 9.2 架构总览

```
                    ┌──────────────┐
                    │   Lead       │
                    │  agent_loop  │
                    └──────┬───────┘
                           │ spawn_teammate / send_message / broadcast
                           ▼
              ┌────────────────────────────┐
              │     MessageBus             │
              │  .team/inbox/{name}.jsonl  │
              └──┬──────────┬──────────┬──┘
                 │          │          │
            ┌────▼───┐ ┌───▼────┐ ┌──▼─────┐
            │ Alice  │ │  Bob   │ │ Carol  │
            │ _loop  │ │ _loop  │ │ _loop  │
            │ Thread │ │ Thread │ │ Thread │
            └───┬────┘ └───┬────┘ └──┬─────┘
                │          │         │
                └──────┬───┘────────┘
                       ▼
              ┌─────────────────┐
              │  .tasks/        │
              │  task_*.json    │
              │  (共享任务板)    │
              └─────────────────┘
```

### 9.3 关键洞察

```
自主性的核心价值：
1. 自我驱动：队友主动扫描任务板寻找工作
2. 并行执行：每个队友在独立线程中自主运行
3. 负载均衡：任务先到先得，自然分散到空闲队友
4. 弹性伸缩：空闲 60 秒自动退出，释放资源

设计原则：
- WORK 阶段专注执行，最多 50 轮
- IDLE 阶段定期轮询，收件箱优先于任务板
- 超时退出避免资源浪费
- 身份重注入确保压缩后角色不丢失
- 线程锁保护共享状态的并发安全
```

---

## 10. 练习

### 练习 1：任务优先级

修改 `scan_unclaimed_tasks()`，在返回列表前按优先级排序（提示：在 task JSON 中添加 `priority` 字段）。

### 练习 2：认领冲突检测

在 `claim_task()` 中添加二次检查：获得锁后先判断任务是否已有 owner，如果有则返回错误而非覆盖。

### 练习 3：角色偏好过滤

修改扫描逻辑，让队友根据自身 role 偏好过滤任务（如 coder 优先认领带有 `"type": "code"` 标签的任务）。

### 练习 4：动态超时

将 `IDLE_TIMEOUT = 60` 改为动态值：如果任务板上还有 pending 任务，延长超时时间；如果完全空闲，缩短超时。

---

## 11. 下一步

现在队友可以自主认领任务了。但所有任务仍在同一目录中执行，多个队友可能互相干扰。

在 [s12：工作树隔离](./s12-worktree-task-isolation.md) 中，我们将学习：
- Git Worktree 机制
- 任务与工作树绑定
- 隔离的执行环境
- 完整的生命周期事件

---

< 上一章：[s10-团队协议](./s10-team-protocols.md) | [目录](./index.md) | 下一章：[s12-工作树隔离](./s12-worktree-task-isolation.md) >
