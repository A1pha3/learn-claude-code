# s09：Agent 团队 —— 从单兵到协作

> **核心口号**：*"任务太大时，委托给队友"*

> **学习目标**：理解多 Agent 协作的价值，掌握持久化队友机制，学会设计团队通信

---

## 学习目标

完成本章后，你将能够：

1. **理解团队 vs 子 Agent** —— 为什么需要持久化的队友
2. **掌握 MessageBus** —— Agent 间的通信机制
3. **实现 TeammateManager** —— 队友生命周期管理
4. **理解 JSONL 邮箱** —— 持久化的消息传递

---

## 1. 从子 Agent 到团队

### 1.1 回顾 s04 的子 Agent

```python
# s04: 子 Agent（用完即弃）
def run_subagent(prompt: str) -> str:
    # 创建新上下文
    sub_messages = [{"role": "user", "content": prompt}]
    # 运行循环
    for _ in range(30):
        response = client.messages.create(...)
        # ...
    # 返回摘要，子 Agent "死亡"
    return summary
```

**特点**：
- **临时性**：用完即弃，没有记忆
- **无状态**：每次都是新的开始
- **单向通信**：只有父→子的请求

### 1.2 团队的改进

```python
# s09: 持久化队友
teammate = spawn(name="alice", role="coder", prompt="你是代码专家")
# teammate 持续运行，有自己的上下文
# 可以接收多次消息
# 有自己的状态和记忆
```

**特点**：
- **持久性**：持续运行，有记忆
- **双向通信**：可以互相发送消息
- **独立上下文**：每个队友有自己的消息历史

### 1.3 使用场景对比

```
子 Agent 适合：
├── 一次性分析任务
├── 需要隔离上下文的查询
└── 临时助手

团队适合：
├── 长期项目开发
├── 角色分工（前端/后端/测试）
├── 需要多轮对话的任务
└── 持续的团队协作
```

---

## 2. 团队架构

### 2.1 架构图

```
                主 Agent (lead)
                │
                │ spawn(name="alice", role="coder")
                ▼
┌─────────────────────────────────────┐
│           .team/                     │
│  ├── config.json                    │
│  │   ├── {"name": "lead", ...}      │
│  │   ├── {"name": "alice", ...}     │
│  │   └── {"name": "bob", ...}       │
│  │                                  │
│  └── inbox/                         │
│      ├── lead.jsonl  ◄───┐          │
│      ├── alice.jsonl     │          │
│      └── bob.jsonl       │          │
└──────────────────────────┼──────────┘
                           │
                    MessageBus (send/read)
                           │
      ┌────────────────────┼────────────────────┐
      │                    │                    │
      ▼                    ▼                    ▼
┌──────────┐         ┌──────────┐         ┌──────────┐
│  alice   │         │   bob    │         │  charlie │
│  循环    │◄───────►│  循环    │◄───────►│  循环    │
│          │  互相发送  │          │  互相发送  │          │
│ 检查邮箱 │  消息     │ 检查邮箱 │  消息     │ 检查邮箱 │
└──────────┘         └──────────┘         └──────────┘
```

### 2.2 通信机制

```
发送消息 (send)：
┌─────────┐              ┌──────────┐
│  alice  │ ─send───────►│ bob.jsonl│
│         │  ("hello")   │          │
└─────────┘              └──────────┘
                              │
                              │ append
                              ▼
                        {"from": "alice",
                         "content": "hello",
                         "timestamp": ...}

读取消息 (read_inbox)：
┌─────────┐              ┌──────────┐
│   bob   │ ◄─read───────│ bob.jsonl│
│         │  inbox       │          │
└─────────┘   返回消息    └──────────┘
                              │
                              │ drain
                              ▼
                        (文件清空)
```

---

## 3. MessageBus 实现

### 3.1 核心类

```python
import json
import time
from pathlib import Path
from typing import List, Dict, Optional

class MessageBus:
    def __init__(self, team_dir: Path):
        self.dir = team_dir / "inbox"
        self.dir.mkdir(parents=True, exist_ok=True)

    def send(
        self,
        sender: str,
        to: str,
        content: str,
        msg_type: str = "message",
        extra: Optional[Dict] = None
    ) -> str:
        """发送消息到收件人的邮箱"""
        msg = {
            "type": msg_type,
            "from": sender,
            "to": to,
            "content": content,
            "timestamp": time.time()
        }

        if extra:
            msg.update(extra)

        # 追加到收件人的邮箱文件
        inbox_path = self.dir / f"{to}.jsonl"
        with open(inbox_path, "a", encoding="utf-8") as f:
            f.write(json.dumps(msg, ensure_ascii=False) + "\n")

        return f"消息已发送到 {to}"

    def read_inbox(self, name: str) -> str:
        """读取并清空邮箱"""
        inbox_path = self.dir / f"{name}.jsonl"

        if not inbox_path.exists():
            return "[]"

        # 读取所有消息
        messages = []
        with open(inbox_path, "r", encoding="utf-8") as f:
            for line in f:
                if line.strip():
                    messages.append(json.loads(line))

        # 清空文件（drain 模式）
        inbox_path.write_text("", encoding="utf-8")

        return json.dumps(messages, indent=2, ensure_ascii=False)

    def broadcast(
        self,
        sender: str,
        content: str,
        exclude: Optional[List[str]] = None
    ) -> str:
        """广播消息给所有队友"""
        exclude = exclude or []
        exclude.append(sender)  # 不发给自己

        # 获取所有邮箱文件
        recipients = []
        for f in self.dir.glob("*.jsonl"):
            name = f.stem  # 文件名（不含 .jsonl）
            if name not in exclude:
                recipients.append(name)

        # 发送消息
        for recipient in recipients:
            self.send(sender, recipient, content)

        return f"广播已发送给 {len(recipients)} 个队友"
```

### 3.2 JSONL 格式的优势

```jsonl
{"from": "alice", "to": "bob", "content": "你好", "timestamp": 1234567890}
{"from": "lead", "to": "bob", "content": "状态更新", "timestamp": 1234567891}
{"from": "charlie", "to": "bob", "content": "代码已提交", "timestamp": 1234567892}
```

**为什么用 JSONL 而不是 JSON 数组？**

| 特性 | JSON 数组 | JSONL |
|------|-----------|-------|
| 追加写入 | 需要重写整个文件 | 直接追加 |
| 并发安全 | 差 | 好（append 原子） |
| 内存占用 | 需要全部加载 | 可以逐行处理 |
| 清空操作 | 覆盖文件 | 覆盖文件 |

---

## 4. TeammateManager 实现

### 4.1 核心类

```python
import threading
import json
from pathlib import Path
from typing import Dict, List

class TeammateManager:
    def __init__(self, team_dir: Path, message_bus: MessageBus):
        self.dir = team_dir
        self.dir.mkdir(exist_ok=True)
        self.bus = message_bus
        self.config_path = self.dir / "config.json"
        self.config = self._load_config()
        self.threads: Dict[str, threading.Thread] = {}

    def _load_config(self) -> Dict:
        """加载团队配置"""
        if self.config_path.exists():
            return json.loads(self.config_path.read_text(encoding="utf-8"))
        return {"members": []}

    def _save_config(self):
        """保存团队配置"""
        self.config_path.write_text(
            json.dumps(self.config, indent=2, ensure_ascii=False),
            encoding="utf-8"
        )

    def _find_member(self, name: str) -> Optional[Dict]:
        """查找队友"""
        for member in self.config["members"]:
            if member["name"] == name:
                return member
        return None

    def _update_member(self, name: str, **kwargs):
        """更新队友信息"""
        member = self._find_member(name)
        if member:
            member.update(kwargs)
            self._save_config()

    def spawn(
        self,
        name: str,
        role: str,
        prompt: str,
        system: Optional[str] = None
    ) -> str:
        """创建新队友"""
        # 检查是否已存在
        if self._find_member(name):
            return f"队友 '{name}' 已存在"

        # 添加到配置
        member = {
            "name": name,
            "role": role,
            "status": "working",
            "created_at": int(time.time())
        }
        self.config["members"].append(member)
        self._save_config()

        # 启动队友循环（在独立线程中）
        thread = threading.Thread(
            target=self._teammate_loop,
            args=(name, role, prompt, system),
            daemon=True
        )
        thread.start()
        self.threads[name] = thread

        return f"队友 '{name}' (角色: {role}) 已创建"

    def _teammate_loop(
        self,
        name: str,
        role: str,
        prompt: str,
        system: Optional[str]
    ):
        """队友的 Agent 循环"""
        messages = [{"role": "user", "content": prompt}]
        max_rounds = 50

        for _ in range(max_rounds):
            # 检查邮箱
            inbox = self.bus.read_inbox(name)
            if inbox != "[]":
                messages.append({
                    "role": "user",
                    "content": f"<inbox>\n{inbox}\n</inbox>"
                })
                messages.append({
                    "role": "assistant",
                    "content": "已阅读收件箱。"
                })

            # LLM 调用
            response = client.messages.create(
                model=MODEL,
                system=system or f"你是 {name}，角色是 {role}。",
                messages=messages,
                tools=CHILD_TOOLS,
                max_tokens=8000
            )
            messages.append({
                "role": "assistant",
                "content": response.content
            })

            # 检查是否继续
            if response.stop_reason != "tool_use":
                break

            # 执行工具
            results = []
            for block in response.content:
                if block.type == "tool_use":
                    handler = TOOL_HANDLERS.get(block.name)
                    if handler:
                        output = handler(**block.input)
                        results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": str(output)[:50000]
                        })

            messages.append({
                "role": "user",
                "content": results
            })

        # 标记为空闲
        self._update_member(name, status="idle")

    def list_all(self) -> str:
        """列出所有队友"""
        if not self.config["members"]:
            return "无队友"

        lines = ["团队成员："]
        for member in self.config["members"]:
            status_emoji = {
                "working": "🔄",
                "idle": "💤",
                "shutdown": "⏹️"
            }
            emoji = status_emoji.get(member["status"], "❓")
            lines.append(f"  {emoji} {member['name']} ({member['role']})")

        return "\n".join(lines)
```

---

## 5. 工具集成

### 5.1 工具定义

```python
TOOLS = [
    # ... 其他工具 ...
    {
        "name": "spawn",
        "description": "创建一个新的队友",
        "input_schema": {
            "type": "object",
            "properties": {
                "name": {
                    "type": "string",
                    "description": "队友名称"
                },
                "role": {
                    "type": "string",
                    "description": "队友角色（如 coder, tester, reviewer）"
                },
                "prompt": {
                    "type": "string",
                    "description": "给队友的初始指令"
                }
            },
            "required": ["name", "role", "prompt"]
        }
    },
    {
        "name": "send",
        "description": "发送消息给队友",
        "input_schema": {
            "type": "object",
            "properties": {
                "to": {"type": "string"},
                "content": {"type": "string"}
            },
            "required": ["to", "content"]
        }
    },
    {
        "name": "broadcast",
        "description": "广播消息给所有队友",
        "input_schema": {
            "type": "object",
            "properties": {
                "content": {"type": "string"}
            },
            "required": ["content"]
        }
    },
    {
        "name": "read_inbox",
        "description": "读取收件箱",
        "input_schema": {
            "type": "object",
            "properties": {},
            "required": []
        }
    },
    {
        "name": "list_team",
        "description": "列出所有队友",
        "input_schema": {
            "type": "object",
            "properties": {},
            "required": []
        }
    },
]
```

### 5.2 注册处理器

```python
TEAM_DIR = Path(".team")
BUS = MessageBus(TEAM_DIR)
TEAMMATES = TeammateManager(TEAM_DIR, BUS)

TOOL_HANDLERS = {
    # ... 其他工具 ...
    "spawn": lambda **kw: TEAMMATES.spawn(
        kw["name"],
        kw["role"],
        kw["prompt"]
    ),
    "send": lambda **kw: BUS.send("lead", kw["to"], kw["content"]),
    "broadcast": lambda **kw: BUS.broadcast("lead", kw["content"]),
    "read_inbox": lambda **kw: BUS.read_inbox("lead"),
    "list_team": lambda **kw: TEAMMATES.list_all(),
}
```

---

## 6. 完整示例

### 6.1 创建团队

```
用户：创建一个开发团队，包含前端和后端开发者

Lead Agent：
  spawn(name="alice", role="前端开发", prompt="负责 React 组件开发")
  → 队友 'alice' (角色: 前端开发) 已创建

  spawn(name="bob", role="后端开发", prompt="负责 API 和数据库")
  → 队友 'bob' (角色: 后端开发) 已创建

  list_team()
  → 团队成员：
     🔄 alice (前端开发)
     🔄 bob (后端开发)
```

### 6.2 团队协作

```
Lead Agent：
  send(to="bob", content="请创建用户 API")
  → 消息已发送到 bob

[Bob 的循环中]
  检查邮箱 → 收到消息
  思考 → 创建 API
  执行 → write_file("api/users.py", ...)
  完成
  更新状态 → idle

Lead Agent：
  send(to="alice", content="用户 API 已完成，请创建前端页面")
  → 消息已发送到 alice

[Alice 的循环中]
  检查邮箱 → 收到消息
  思考 → 创建页面
  执行 → write_file("src/Users.jsx", ...)
  完成
```

---

## 7. 与 s04 的对比

| 特性 | s04 子 Agent | s09 团队 |
|------|-------------|---------|
| 生命周期 | 临时（用完即弃） | 持久（持续运行） |
| 上下文 | 每次全新 | 独立累积 |
| 通信 | 单向（父→子） | 双向（互发消息） |
| 状态 | 无 | 有（working/idle） |
| 记忆 | 无 | 有（消息历史） |
| 适用场景 | 临时任务 | 长期协作 |

---

## 8. 常见问题

### Q1：队友会无限运行吗？

**答案**：不会。每个队友循环有最大轮次限制（如 50 轮），之后自动转为 `idle` 状态。

### Q2：如何停止队友？

```python
def shutdown(self, name: str) -> str:
    """停止队友"""
    member = self._find_member(name)
    if not member:
        return f"队友不存在：{name}"

    if member["status"] == "shutdown":
        return f"队友已停止：{name}"

    # 发送停止消息
    self.bus.send("lead", name, "请停止工作", "shutdown")
    self._update_member(name, status="shutdown")

    return f"已请求停止队友：{name}"
```

### Q3：队友可以互相通信吗？

**答案**：可以。任何队友都可以调用 `send` 工具向其他队友发送消息。

```python
# Alice 的循环中
send(to="bob", content="API 规范已更新，请查看")
```

### Q4：如何处理消息丢失？

**答案**：JSONL 的追加操作是原子的，不容易丢失。但如果需要可靠性，可以实现确认机制：

```python
# 发送时带 ID
msg_id = str(uuid.uuid4())
self.send(sender, to, content, extra={"msg_id": msg_id})

# 接收后确认
self.acknowledge(msg_id)
```

---

## 9. 小结

### 9.1 核心要点

| 要点 | 说明 |
|------|------|
| **持久化队友** | 持续运行，有独立上下文 |
| **MessageBus** | JSONL 邮箱，支持双向通信 |
| **线程隔离** | 每个队友在独立线程中运行 |
| **状态管理** | working/idle/shutdown |

### 9.2 代码模板

```python
# 创建队友
spawn(name="alice", role="coder", prompt="...")

# 发送消息
send(to="alice", content="请实现用户登录")

# 广播
broadcast(content="第一阶段完成")

# 读取收件箱
read_inbox()

# 列出团队
list_team()
```

### 9.3 关键洞察

```
团队的核心价值：
1. 持久化：队友持续运行，有记忆
2. 通信：队友之间可以互相发送消息
3. 隔离：每个队友有独立的上下文
4. 协作：可以分工完成复杂任务

设计原则：
- JSONL 邮箱：持久化消息传递
- 独立线程：每个队友并发运行
- 状态管理：tracking working/idle/shutdown
- 配置文件：记录团队成员信息
```

---

## 10. 练习

### 练习 1：角色模板

定义常见角色的预设模板（coder、tester、reviewer），简化队友创建。

### 练习 2：消息确认

实现消息确认机制，确保消息被接收。

### 练习 3：队友发现

队友可以自动发现其他队友，获取团队列表。

### 练习 4：状态同步

实现心跳机制，定期同步队友状态。

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
