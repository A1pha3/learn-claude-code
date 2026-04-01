# s09：Agent 团队 —— 从单兵到协作

> **核心口号**：*"任务太大时，委托给队友"*

> **学习目标**：理解多 Agent 协作的价值，掌握持久化队友机制，学会设计团队通信

---

## 学习目标

完成本章后，你将能够：

1. **理解团队 vs 子 Agent** —— 为什么需要持久化的队友
2. **掌握 MessageBus** —— 基于 JSONL 邮箱的异步通信机制
3. **实现 TeammateManager** —— 队友生命周期管理（spawn → work → idle → shutdown）
4. **理解 JSONL 邮箱** —— 持久化的、append-only 的消息传递

**学习分层表**：

| 层级 | 目标 | 检验标准 |
|------|------|----------|
| ⭐ 基础 | 理解团队 vs 子 Agent 的区别 | 能说出持久化队友的三个特点（独立上下文、双向通信、状态管理） |
| ⭐⭐ 进阶 | 独立搭建多人协作团队 | 能完成 spawn → send → drain 闭环，理解 JSONL 邮箱的 drain 语义与 append 原子性 |
| ⭐⭐⭐ 专家 | 设计角色分工与通信协议 | 能为具体项目定义角色模板，设计通信拓扑，评估 spawn 权限隔离的必要性 |

---

## 0. 上手演练（建议先做）

先运行团队协作示例，再看架构与通信协议：

```bash
uv run python agents/s09_agent_teams.py
```

建议观察：

1. 队友创建后是否保持独立上下文与持续状态。
2. 消息在 MessageBus 与 JSONL 邮箱中的流转过程。
3. 与 s04 子 Agent 相比，团队模式在哪些场景更高效。

### 本章完成标准

- 能判断一个任务应该用子 Agent 还是持久化团队。
- 能独立搭建"创建队友—发消息—收结果"的最小闭环。
- 能解释 JSONL 邮箱在可恢复与审计方面的作用。

---

## 1. 从子 Agent 到团队

### 1.1 回顾 s04 的子 Agent

```python
# s04: 子 Agent（用完即弃）
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]
    for _ in range(30):
        response = client.messages.create(...)
        # ...
    return summary  # 返回摘要，子 Agent "死亡"
```

**特点**：
- **临时性**：用完即弃，没有记忆
- **无状态**：每次都是新的开始
- **单向通信**：只有父→子的请求

### 1.2 团队的改进

```python
# s09: 持久化队友
TEAM.spawn(name="alice", role="coder", prompt="你是代码专家")
# alice 持续运行在自己的线程中
# 有自己的上下文和消息历史
# 可以通过邮箱收发消息
# 完成任务后进入 idle，等待新任务
```

**特点**：
- **持久性**：持续运行，有独立上下文和记忆
- **双向通信**：队友之间可以互相发送消息
- **状态管理**：working / idle / shutdown

### 1.3 使用场景对比

| 特性 | 子 Agent（s04） | 团队队友（s09） |
|------|----------------|----------------|
| 生命周期 | 临时（用完即弃） | 持久（持续运行） |
| 上下文 | 每次全新 | 独立累积 |
| 通信 | 单向（父→子） | 双向（互发消息） |
| 状态 | 无 | working / idle / shutdown |
| 适用场景 | 一次性分析、临时助手 | 长期项目、角色分工、多轮协作 |

---

## 2. 团队架构

### 2.1 整体架构

```
s09 新增组件（标记 ★）

.team/                              主 Agent (lead)
├── ★config.json                    │
│   ├── {"team_name": "default"}    │ ★spawn_teammate(name="alice", ...)
│   ├── {"name":"alice",            ▼
│   │    "role":"coder",        ┌──────────┐
│   │    "status":"idle"}       │★config   │
│   └── {"name":"bob",...}      │  .json   │
│                               └──────────┘
└── ★inbox/                          │
    ├── lead.jsonl              spawn → 写入 config + 启动线程
    ├── alice.jsonl                 │
    └── bob.jsonl              ┌────┴────┐
                               │         │
                          ┌────▼───┐ ┌───▼────┐
                          │ alice  │ │  bob   │
                          │ 线程   │ │ 线程   │
                          │ agent  │ │ agent  │
                          │ loop   │ │ loop   │
                          └────────┘ └────────┘
                               ↑         ↑
                               │★JSONL   │
                               └─邮箱────┘

★ MessageBus（s09 核心新组件）
├── send(sender, to, content) → append-only 写入 JSONL
├── read_inbox(name) → drain 模式读取 + 清空
└── broadcast(sender, content, teammates) → 逐个 send

★ TeammateManager（s09 核心新组件）
├── spawn(name, role, prompt) → 写入 config.json + 启动 daemon 线程
├── _teammate_loop(name, role, prompt) → 独立 agent 循环（最多 50 轮）
├── list_all() → 读取 config.json 显示成员状态
└── send_message / read_inbox → 代理 MessageBus 操作
```

### 2.2 目录结构（源码实际布局）

源码中的目录结构（agents/s09_agent_teams.py 第 62-63 行）：

```python
TEAM_DIR = WORKDIR / ".team"
INBOX_DIR = TEAM_DIR / "inbox"
```

```
.team/
├── config.json           # 团队成员配置
│   {
│     "team_name": "default",
│     "members": [
│       {"name": "alice", "role": "coder", "status": "working"},
│       {"name": "bob", "role": "tester", "status": "idle"}
│     ]
│   }
└── inbox/                # JSONL 邮箱目录
    ├── lead.jsonl        # Lead 的收件箱
    ├── alice.jsonl       # Alice 的收件箱
    └── bob.jsonl         # Bob 的收件箱
```

---

## 3. MessageBus 实现

> **概念速查**
>
> | 概念 | 定义 | 源码映射 |
> |------|------|----------|
> | **MessageBus** | 团队通信中枢：基于 JSONL 邮箱实现 Agent 间异步消息传递 | `class MessageBus` |
> | **JSONL mailbox** | 持久化邮箱：每个 Agent 一个 `.jsonl` 文件，append-only 写入，drain 读取 | `inbox/{name}.jsonl` |
> | **drain semantics** | 读取后清空——`read_inbox()` 读取全部消息后立即 `write_text("")` 清空文件 | `read_inbox()` 中 read + clear |
> | **config.json lifecycle** | 团队配置的创建→更新→恢复周期：spawn 时写入、idle 时更新、重启时读取恢复 | `_load_config()` / `_save_config()` |

### 3.1 核心类（源码分析）

### 3.1 核心类（源码分析）

MessageBus 是团队通信的核心。源码位于 `agents/s09_agent_teams.py` 第 77-118 行：

```python
class MessageBus:
    def __init__(self, inbox_dir: Path):
        self.dir = inbox_dir
        self.dir.mkdir(parents=True, exist_ok=True)
```

### 3.2 消息格式

每条消息是一个 JSON 对象，包含以下字段：

```json
{
  "type": "message",
  "from": "lead",
  "content": "请实现用户登录功能",
  "timestamp": 1730000000.123
}
```

**消息类型**（源码第 67-73 行定义了 5 种类型）：

| 类型 | 用途 | 引入章节 |
|------|------|----------|
| `message` | 普通文本消息 | s09 |
| `broadcast` | 广播给所有队友 | s09 |
| `shutdown_request` | 请求优雅关机 | s10 |
| `shutdown_response` | 响应关机请求 | s10 |
| `plan_approval_response` | 计划审批响应 | s10 |

### 3.3 发送消息（append-only）

```python
def send(self, sender: str, to: str, content: str,
         msg_type: str = "message", extra: dict = None) -> str:
    if msg_type not in VALID_MSG_TYPES:
        return f"Error: Invalid type '{msg_type}'. Valid: {VALID_MSG_TYPES}"
    msg = {"type": msg_type, "from": sender, "content": content, "timestamp": time.time()}
    if extra:
        msg.update(extra)
    # 关键：append-only 写入，保证原子性
    inbox_path = self.dir / f"{to}.jsonl"
    with open(inbox_path, "a") as f:
        f.write(json.dumps(msg) + "\n")
    return f"Sent {msg_type} to {to}"
```

**设计要点**：
1. **验证消息类型**：只接受预定义的 5 种类型
2. **append-only**：`open(path, "a")` 追加写入，操作是原子的
3. **extra 字段**：支持附加自定义数据（如 request_id）

### 3.4 读取邮箱（drain 模式）

```python
def read_inbox(self, name: str) -> list:
    inbox_path = self.dir / f"{name}.jsonl"
    if not inbox_path.exists():
        return []
    messages = []
    for line in inbox_path.read_text().strip().splitlines():
        if line:
            messages.append(json.loads(line))
    # 关键：读取后清空（drain 模式）
    inbox_path.write_text("")
    return messages
```

**drain 模式的含义**：读取邮箱后立即清空，确保同一条消息不会被重复处理。

### 3.5 广播

```python
def broadcast(self, sender: str, content: str, teammates: list) -> str:
    count = 0
    for name in teammates:
        if name != sender:  # 不发给自己
            self.send(sender, name, content, "broadcast")
            count += 1
    return f"Broadcast to {count} teammates"
```

### 3.6 JSONL 格式的优势

```jsonl
{"type": "message", "from": "alice", "content": "你好", "timestamp": 1730000000}
{"type": "broadcast", "from": "lead", "content": "阶段 1 完成", "timestamp": 1730000001}
```

| 特性 | JSON 数组 | JSONL |
|------|-----------|-------|
| 追加写入 | 需要重写整个文件 | 直接 append（原子操作） |
| 并发安全 | 差（需要锁整个文件） | 好（append 原子性） |
| 内存占用 | 需要全部加载 | 可以逐行处理 |
| 可恢复性 | 文件损坏 = 全部丢失 | 单行损坏不影响其他行 |

---

## 4. TeammateManager 实现

### 4.1 核心类

TeammateManager 管理队友的整个生命周期。源码位于第 123-247 行：

```python
class TeammateManager:
    def __init__(self, team_dir: Path):
        self.dir = team_dir
        self.config_path = self.dir / "config.json"
        self.config = self._load_config()  # 从磁盘加载
        self.threads = {}  # name → Thread 映射
```

### 4.2 队友生命周期

```
spawn → working → idle → working → idle → ... → shutdown
  │                                                  │
  │ 创建线程                                         │ 超过 50 轮或异常
  ▼                                                  ▼
config.json:                     config.json:
{"status": "working"}            {"status": "idle"}
                                 (如果被请求关机则为 "shutdown")
```

### 4.3 spawn 实现

```python
def spawn(self, name: str, role: str, prompt: str) -> str:
    # 1. 检查是否已存在
    member = self._find_member(name)
    if member:
        if member["status"] not in ("idle", "shutdown"):
            return f"Error: '{name}' is currently {member['status']}"
        member["status"] = "working"  # 重新激活
    else:
        member = {"name": name, "role": role, "status": "working"}
        self.config["members"].append(member)

    # 2. 持久化到磁盘
    self._save_config()

    # 3. 启动守护线程
    thread = threading.Thread(
        target=self._teammate_loop,
        args=(name, role, prompt),
        daemon=True,  # 守护线程：主程序退出时自动结束
    )
    self.threads[name] = thread
    thread.start()
    return f"Spawned '{name}' (role: {role})"
```

**关键设计**：
- 如果队友已存在且为 idle/shutdown，可以重新激活
- 使用 `daemon=True` 确保主程序退出时线程自动终止
- 配置立即持久化到 `config.json`

### 4.4 队友的 Agent 循环

```python
def _teammate_loop(self, name: str, role: str, prompt: str):
    sys_prompt = (
        f"You are '{name}', role: {role}, at {WORKDIR}. "
        f"Use send_message to communicate. Complete your task."
    )
    messages = [{"role": "user", "content": prompt}]
    tools = self._teammate_tools()
    for _ in range(50):  # 最多 50 轮
        # 1. 检查邮箱
        inbox = BUS.read_inbox(name)
        for msg in inbox:
            messages.append({"role": "user", "content": json.dumps(msg)})

        # 2. 调用 LLM（源码第 176-185 行有 try/except 保护）
        try:
            response = client.messages.create(
                model=MODEL, system=sys_prompt,
                messages=messages, tools=tools, max_tokens=8000,
            )
        except Exception:
            break  # LLM 调用失败（网络错误、额度耗尽等），直接跳出循环
        messages.append({"role": "assistant", "content": response.content})

        # 3. 检查是否继续
        if response.stop_reason != "tool_use":
            break

        # 4. 执行工具
        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = self._exec(name, block.name, block.input)
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(output)})
        messages.append({"role": "user", "content": results})

    # 5. 循环结束，标记为 idle
    member = self._find_member(name)
    if member and member["status"] != "shutdown":
        member["status"] = "idle"
        self._save_config()
```

**关于 `_save_config()` 的持久化风险**：`_save_config()` 的实现（源码第 136-137 行）是 `self.config_path.write_text(json.dumps(...))`，即每次调用都完整覆盖 config.json 文件。如果在写入过程中程序崩溃（如断电、进程被 kill），可能导致文件内容不完整或为空。在生产环境中，更安全的做法是"先写入临时文件，再原子重命名"（先 `write_text` 到 `.tmp` 文件，然后 `os.replace` 覆盖目标文件），以确保要么完全写入成功，要么保持旧文件不变。

### 4.5 队友的工具集

队友的工具集与 Lead 不同。源码第 221-236 行：

**与 Lead 的工具对比**：

| 工具 | Lead | Teammate |
|------|------|----------|
| bash / read_file / write_file / edit_file | ✅ | ✅ |
| send_message | ✅ | ✅ |
| read_inbox | ✅ | ✅ |
| broadcast | ✅ | ❌ |
| spawn_teammate | ✅ | ❌ |
| list_teammates | ✅ | ❌ |

**为什么队友不能 spawn？** 防止无限递归：alice spawn bob，bob spawn charlie，charlie spawn dave...

---

## 5. Lead 的 Agent 循环

### 5.1 与标准循环的区别

Lead 的循环在标准 Agent 循环基础上增加了**邮箱检查**（源码第 344-381 行）：

```python
def agent_loop(messages: list):
    while True:
        # 1. 每轮循环前检查邮箱
        inbox = BUS.read_inbox("lead")
        if inbox:
            messages.append({
                "role": "user",
                "content": f"<inbox>{json.dumps(inbox, indent=2)}</inbox>",
            })
            messages.append({
                "role": "assistant",
                "content": "Noted inbox messages.",
            })

        # 2. 标准 LLM 调用 + 工具执行（与 s01 相同）
        response = client.messages.create(...)
        # ...
```

**与 s01 的核心循环对比**：唯一的区别是在每轮循环开始前增加了邮箱检查和消息注入。

### 5.2 交互命令

源码支持两个快捷命令（第 393-397 行）：

- `/team`：查看团队状态（等价于 `list_teammates()`）
- `/inbox`：查看 Lead 的邮箱

---

## 6. 完整示例

### 6.1 创建团队

```
用户：创建一个开发团队，包含前端和后端开发者

Lead Agent：
  spawn_teammate(name="alice", role="前端开发", prompt="负责 React 组件开发")
  → Spawned 'alice' (role: 前端开发)

  spawn_teammate(name="bob", role="后端开发", prompt="负责 API 和数据库")
  → Spawned 'bob' (role: 后端开发)

  list_teammates()
  → Team: default
    alice (前端开发): working
    bob (后端开发): working
```

### 6.2 消息流转示意

```
Lead                        Alice                      Bob
  │                           │                          │
  │─send_message("bob","...")─┼─────────────────────────→│
  │                           │                          │
  │                           │              Bob 循环中检查邮箱
  │                           │              收到消息，执行任务
  │                           │                          │
  │─send_message("alice",".")→│                          │
  │                           │                          │
  │              Alice 循环中检查邮箱                     │
  │              收到消息，执行任务                       │
  │                           │                          │
  │─read_inbox() ←────────────┤ ←─── send_message ──────┤
  │  收到两人的完成通知          │                          │
```

---

## 7. 与 s04 的对比

| 特性 | s04 子 Agent | s09 团队 | 演进动机 |
|------|-------------|---------|----------|
| 生命周期 | 临时（用完即弃） | 持久（持续运行） | 复杂项目需要角色持续在线，频繁创建销毁开销大且丢失上下文 |
| 上下文 | 每次全新 | 独立累积 | 多轮协作需要记住之前的讨论结果和代码风格偏好 |
| 通信 | 单向（父→子） | 双向（互发消息） | 团队成员间需直接沟通（如后端通知前端接口就绪），不应事事通过 Lead 中转 |
| 状态 | 无 | working / idle / shutdown | Lead 需要了解队友实时状态以分配任务，无状态则无法判断谁可用 |
| 记忆 | 无 | 有（消息历史） | 队友处理过的问题不应重复处理，消息历史避免重复劳动 |
| 存储方式 | 无 | config.json + JSONL 邮箱 | 持久化存储支持进程重启后恢复团队状态，JSONL append 保证写入原子性 |

### 7.1 设计决策记录

| 决策 | 选择 | 原因 | 替代方案 |
|------|------|------|----------|
| 通信介质 | JSONL 文件（append-only） | 追加写入是原子操作，并发安全，单行损坏不影响整体 | JSON 数组文件：追加需重写全部内容；内存队列：进程退出后丢失 |
| 邮箱读取 | drain 模式（读取后清空） | 保证消息不重复处理，与 s08 的 drain_notifications 思路一致 | 保留已读标记：增加复杂度，需维护 read/unread 状态 |
| 团队配置存储 | 单一 config.json | 成员数量有限（通常 < 10），单文件读写简单 | 每个成员独立配置文件：过度工程，增加一致性维护负担 |
| 队友工具隔离 | 队友不含 spawn/broadcast | 防止无限递归（alice spawn bob，bob spawn charlie...） | 信任模型：允许任意 spawn，通过层级限制控制深度 |
| 线程模型 | daemon=True 守护线程 | 与 s08 相同的理由：队友线程不应阻止用户退出 | 可中断线程：需要实现优雅关机协议（s10 会引入） |
| 队友循环上限 | 50 轮 | 防止 token 无限消耗，50 轮足以完成大多数单次任务 | 无限制：成本失控风险；10 轮太少：复杂任务无法完成 |
| 消息类型 | 预定义枚举 + extra 字段 | 类型安全（拒绝未知类型），extra 提供扩展灵活性 | 自由格式：缺少约束，易出错；严格 schema：过度限制 |
| config.json 写入 | `write_text()` 全量覆盖 | 成员少、写入频率低，简单直接 | 原子重命名（write_to_tmp + os.replace）：更安全但复杂，教学场景不必要 |

---

## 8. 常见问题

### Q1：队友会无限运行吗？

**不会**。每个队友循环有最大轮次限制（50 轮），之后自动转为 `idle` 状态。

### Q2：如何重新激活 idle 的队友？

调用 `spawn_teammate` 即可，TeammateManager 会检测到队友已存在且状态为 idle，自动重新激活。

### Q3：队友可以互相通信吗？

**可以**。每个队友都有 `send_message` 工具，可以向任何队友发送消息。

### Q4：如何处理消息丢失？

JSONL 的 append 操作是原子的，不容易丢失。但如果需要更高可靠性，可以实现确认机制（s10 会引入结构化的请求-响应协议）。

---

## 实战应用：为你的项目设计 Agent 团队

### 场景一：Web 开发团队

```
Lead (协调者)
 ├─ Alice (前端开发)
 │   role: frontend
 │   prompt: "你负责 React 组件开发和页面交互。
 │            收到设计稿后先实现静态页面，再接入 API。"
 │   通信：接收 Lead 的任务分配，完成后通过 send_message 回报
 │
 ├─ Bob (后端开发)
 │   role: backend
 │   prompt: "你负责 API 端点和数据库设计。
 │            先完成 API 契约，再实现业务逻辑。"
 │   通信：接收需求 → 设计 API → 回报契约 → 实现 → 通知 Alice 接口就绪
 │
 └─ Carol (测试)
     role: tester
     prompt: "你负责编写和执行测试。
              收到功能完成通知后，先写测试用例再执行。"
     通信：监听 Alice/Bob 的完成消息 → 自动编写测试 → 汇报结果
```

通信拓扑：Lead 分配任务给 Alice 和 Bob；Bob 完成后通知 Alice（接口就绪）；Alice 和 Bob 都完成后通知 Carol。

### 场景二：数据 ETL 流水线

```
Lead → Collector (采集 Agent)
          ↓ send_message("原始数据已就绪")
       Cleaner (清洗 Agent)
          ↓ send_message("清洗完成，已入库")
       Analyst (分析 Agent)
          ↓ send_message("分析报告已生成")
       Reporter (报告 Agent)
```

这是一个链式管道：每个 Agent 完成阶段工作后通知下游。角色定义示例：

```python
spawn_teammate(name="collector", role="data-collector",
    prompt="从 CSV/API/DB 采集原始数据，存入 .data/raw/ 目录。完成后通知 cleaner。")

spawn_teammate(name="cleaner", role="data-cleaner",
    prompt="清洗 .data/raw/ 中的数据：去重、填充缺失值、格式标准化。完成后通知 analyst。")
```

### 场景三：安全审计团队

```
Lead (安全负责人)
 ├─ Scanner (漏洞扫描)
 │   prompt: "运行 OWASP ZAP 和 npm audit，输出漏洞清单"
 │
 ├─ PenTester (渗透测试)
 │   prompt: "根据扫描结果执行手动渗透测试，记录攻击路径"
 │
 ├─ Reporter (报告编写)
 │   prompt: "汇总扫描和渗透结果，生成风险评级报告"
 │
 └─ Verifier (修复验证)
     prompt: "收到修复完成通知后，重新测试确认漏洞已修复"
```

通信设计：Scanner 完成 → Lead 分发给 PenTester；PenTester 和 Scanner 结果汇总给 Reporter；开发修复后 Lead 通知 Verifier 复测。

### "从子 Agent 升级到团队"决策矩阵

| 评估维度 | 用子 Agent (s04) | 用团队 (s09) |
|----------|------------------|--------------|
| 任务持续性 | 一次性分析、单次转换 | 长期项目、需要多轮协作 |
| 上下文需求 | 无需记忆，每次独立 | 需要累积经验（如代码风格偏好） |
| 通信模式 | 父→子单向即可 | 需要双向沟通或队友间直接通信 |
| 并发数量 | 1-2 个临时子任务 | 3+ 个角色持续运行 |
| 状态管理 | 不需要跟踪状态 | 需要知道谁在做什么（working/idle） |
| 故障恢复 | 不需要，重做即可 | 需要从 JSONL 邮箱和 config.json 恢复 |

---

## 9. 小结

### 9.1 核心要点

| 要点 | 说明 |
|------|------|
| **持久化队友** | 持续运行在独立线程中，有独立上下文和记忆 |
| **MessageBus** | 基于 JSONL 邮箱的异步通信，append-only + drain 模式 |
| **config.json** | 持久化团队成员信息，支持重启后恢复 |
| **工具隔离** | 队友不能 spawn 新队友，防止无限递归 |

### 9.2 关键洞察

```
团队的核心价值：
1. 持久化：队友持续运行，有独立的上下文记忆
2. 通信：通过 JSONL 邮箱实现异步消息传递
3. 隔离：每个队友有独立的线程和消息历史
4. 协作：可以分工完成复杂任务

设计原则：
- JSONL 邮箱：append 原子性 + drain 语义
- 守护线程：daemon=True 确保不阻塞主程序退出
- 工具隔离：队友不含 spawn，防止递归
- 配置持久化：config.json 跨会话记录状态
```

---

## 10. 练习

### 练习 1：角色模板

定义常见角色的预设模板（coder、tester、reviewer），简化队友创建。

提示：可以定义一个 `ROLE_TEMPLATES` 字典，将角色名映射到预设的 role 描述和默认 prompt 模板。在 `spawn` 方法中查找模板，如果匹配则自动填充 role 和 prompt 参数。

**验收标准**：
- 定义 `ROLE_TEMPLATES = {"coder": {...}, "tester": {...}, "reviewer": {...}}`
- `spawn_teammate(name="alice", role="coder")` 无需指定 prompt 即可使用预设模板
- 预设模板中的 prompt 包含具体行为指引（如 tester 的 prompt 包含"先写测试用例再执行"）
- 自定义 prompt 优先级高于模板——`spawn_teammate(name="alice", role="coder", prompt="自定义指令")` 使用自定义版本

### 练习 2：消息确认

实现消息确认机制，确保消息被接收和处理。

提示：可以在 MessageBus 中添加 `confirm` 消息类型，发送方在消息中附带一个唯一 ID，接收方处理完后通过 `send` 回复一条带该 ID 的确认消息。发送方在 `read_inbox` 时检查是否收到对应的确认。

**验收标准**：
- 发送消息时自动附带 `"msg_id": str(uuid.uuid4())[:8]` 字段
- 接收方处理消息后自动回复一条 `"type": "confirm", "original_msg_id": "..."` 消息
- 发送方可以调用 `BUS.is_confirmed(msg_id)` 检查某条消息是否已被确认
- 超过 30 秒未确认的消息可以通过 `BUS.get_unconfirmed(sender)` 查询

### 练习 3：队友发现

队友可以自动发现其他队友，获取团队列表。

提示：可以为队友添加一个 `list_teammates` 工具，内部调用 `TEAM.list_all()` 读取 config.json 中的成员列表。注意不要暴露 spawn 等管理能力，只提供只读查询。

**验收标准**：
- 队友的工具集新增 `list_teammates` 工具
- 队友调用 `list_teammates` 后能看到所有成员的 name、role 和 status
- 队友无法通过此工具获取 spawn 等管理能力（只读）
- 输出格式与 Lead 的 `list_teammates` 一致

### 练习 4：状态同步

实现心跳机制，定期同步队友状态。

提示：可以在 `_teammate_loop` 中每隔 N 轮调用一次 `_save_config()` 将当前状态写入 config.json，同时 Lead 端在 `list_teammates` 中对比内存状态与文件状态，检测"僵尸"队友（线程已退出但 config 中仍为 working）。

**验收标准**：
- 队友每 10 轮自动调用 `_save_config()` 更新 config.json 中的 `"last_heartbeat"` 时间戳
- Lead 的 `list_teammates` 输出中增加心跳时间显示（如 `alice (coder): working [heartbeat: 3s ago]`）
- Lead 检测到"僵尸队友"（config 中 status 为 working 但心跳超过 60 秒）时标记为 `[STALE]`
- 检测到僵尸队友后自动将其状态更新为 `"idle"` 并写入 config.json

---

## 11. 下一步

现在团队可以协作了，但队友之间缺乏结构化的协商机制。如何优雅地处理关机、计划审批等场景？

在 [s10：团队协议](./s10-team-protocols.md) 中，我们将学习：
- 请求-响应模式
- 状态机设计
- 关机握手协议
- 计划审批流程

---

< 上一章：[s08-后台任务](./s08-background-tasks.md) | [目录](./index.md) | 下一章：[s10-团队协议](./s10-team-protocols.md) >
