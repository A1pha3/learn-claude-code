# s10：团队协议 —— 协作的艺术

> **核心口号**：*"队友需要共享通信规则"*

---

## 学习目标

完成本章后，你将能够：

1. **理解为什么需要协议** —— 自由通信的问题
2. **掌握请求-响应模式** —— 通用的协商机制（request_id 关联）
3. **理解状态跟踪器** —— `shutdown_requests` / `plan_requests` 字典 + `_tracker_lock` 线程安全
4. **掌握关机协议的完整握手流程** —— request -> response -> 状态迁移
5. **掌握计划审批的实际代码路径** —— plan_approval 工具 -> lead 审批 -> 通知队友
6. **应用协议到多个场景** —— 同一个 FSM 模式，两个不同领域

**学习分层表**：

| 层级 | 目标 | 检验标准 |
|------|------|----------|
| ⭐ 基础 | 理解协议的必要性 | 能解释 request_id 的作用，能画出 pending -> approved/rejected 状态机 |
| ⭐⭐ 进阶 | 独立实现新的请求-响应协议 | 能复用 FSM 模式添加新跟踪器（如 vote_requests），理解 `_tracker_lock` 的保护范围 |
| ⭐⭐⭐ 专家 | 设计多协议协同与防御性通信 | 能处理协议嵌套、超时兜底、死锁预防，能评估同名工具在 Lead/队友端的语义差异 |

---

## 0. 上手演练（建议先做）

先运行团队协议示例，再阅读请求-响应状态机：

```bash
uv run python agents/s10_team_protocols.py
```

建议观察：

1. 同一 `request_id` 如何贯穿请求与响应。
2. `pending -> approved/rejected` 状态迁移何时发生。
3. 协议化通信如何降低"误停机、误执行"的协作风险。

---

## 1. 问题的本质

### 1.1 自由通信的问题

在 s09 中，队友可以自由发送消息，但缺乏结构化的协商机制：

```
Lead -> Bob: "请停止工作"
Bob [立即停止，文件可能损坏！]
```

**问题**：
1. **没有协商**：Bob 可能正在写入重要文件
2. **没有确认**：Lead 不知道 Bob 是否真的停止了
3. **没有理由**：Bob 不知道为什么要停止
4. **没有关联**：多个请求并发时无法区分

### 1.2 协议化的通信

```
Lead -> Bob: shutdown_request {request_id: "a1b2c3d4", reason: "计划变更"}
Bob 收到请求 → LLM 决策 → 完成当前工作
Bob -> Lead: shutdown_response {request_id: "a1b2c3d4", approve: true}
Bob [清理，退出]
Lead [收到确认，更新状态]
```

**四个核心改进**：
1. **显式请求** —— 标准化格式
2. **request_id** —— 关联请求和响应（UUID 前 8 位）
3. **协商** —— 被请求者可以同意或拒绝（LLM 自主决策）
4. **确认** —— 请求者知道最终结果

---

## 2. 请求-响应模式

### 2.1 通用模式

```
请求者                            被请求者
  │                                  │
  │─── request {req_id, ...} ───────>│
  │                                  │
  │                    处理请求      │
  │                   做出决定       │
  │                                  │
  │<── response {req_id, ...} ──────┤
  │                                  │
  收到结果，更新状态
```

### 2.2 状态机（FSM）

```
        ┌─────────────┐
        │   pending   │ ◄──── 创建请求时初始化
        └──────┬──────┘
               │
      ┌────────┴────────┐
      │                 │
   approve          reject
      │                 │
      ▼                 ▼
┌──────────┐     ┌──────────┐
│ approved │     │ rejected │
└──────────┘     └──────────┘
```

s10 中有两个独立的状态跟踪器，各自运行同一套 FSM：

| 跟踪器 | 字典变量 | 请求方 | 响应方 | 含义 |
|--------|----------|--------|--------|------|
| 关机协议 | `shutdown_requests` | Lead | Teammate | Lead 请求队友关机，队友决定是否同意 |
| 计划审批 | `plan_requests` | Teammate | Lead | 队友提交计划，Lead 决定是否批准 |

### 2.3 线程安全：_tracker_lock

源码中所有对跟踪器字典的读写都通过 `_tracker_lock` 保护：

```python
_tracker_lock = threading.Lock()
```

这是因为 `shutdown_requests` 和 `plan_requests` 是全局字典，被多个线程（Lead 的主线程 + 各队友线程）并发访问。`_tracker_lock` 确保状态更新是原子的。

---

## 3. 关机协议

### 3.1 问题场景

```
场景 1：粗暴关机（s09 的局限）
Lead -> Bob: "停止！"
Bob [立即终止]
结果：文件损坏，状态丢失

场景 2：优雅关机（s10 的协议）
Lead -> Bob: shutdown_request {request_id: "a1b2c3d4"}
Bob [完成当前文件]
Bob [通过 shutdown_response 工具回复 approve=True]
Bob [循环检测 should_exit，退出线程]
Lead [通过 shutdown_response 工具查看状态：approved]
结果：干净退出
```

### 3.2 源码分析：请求端（Lead）

`handle_shutdown_request()` 函数（第 350-358 行）处理 Lead 发起的关机请求：

```python
def handle_shutdown_request(teammate: str) -> str:
    req_id = str(uuid.uuid4())[:8]          # UUID 前 8 位作为 request_id
    with _tracker_lock:
        shutdown_requests[req_id] = {
            "target": teammate,             # 目标队友名
            "status": "pending"             # 初始状态
        }
    BUS.send(
        "lead", teammate,
        "Please shut down gracefully.",     # 消息内容
        "shutdown_request",                 # 消息类型
        {"request_id": req_id},             # extra 字段，注入 request_id
    )
    return f"Shutdown request {req_id} sent to '{teammate}' (status: pending)"
```

**request_id 的生成**：`str(uuid.uuid4())[:8]` 取 UUID4 前 8 位十六进制字符（约 40 亿种组合，单次运行碰撞概率极低）。

**实际发送的消息**（写入队友的 JSONL 文件）：

```json
{"type": "shutdown_request", "from": "lead", "content": "Please shut down gracefully.", "timestamp": 1712000000.123, "request_id": "a1b2c3d4"}
```

`request_id` 通过 `extra` 参数注入到消息对象中（`msg.update(extra)`）。

### 3.3 源码分析：响应端（队友）

队友的 `_exec()` 方法中处理 `shutdown_response` 工具调用（第 235-245 行）：

```python
if tool_name == "shutdown_response":
    req_id = args["request_id"]
    approve = args["approve"]
    with _tracker_lock:
        if req_id in shutdown_requests:
            shutdown_requests[req_id]["status"] = "approved" if approve else "rejected"
    BUS.send(
        sender, "lead", args.get("reason", ""),
        "shutdown_response",
        {"request_id": req_id, "approve": approve},
    )
    return f"Shutdown {'approved' if approve else 'rejected'}"
```

**关键细节**：
1. **状态更新用锁保护** —— `with _tracker_lock` 确保原子性
2. **请求存在性检查** —— `if req_id in shutdown_requests`：request_id 不存在时静默忽略（不报错，不更新状态）
3. **发送确认给 Lead** —— 通过 `BUS.send()` 将决定发回 Lead 收件箱

**队友循环中的退出检测**（第 213-214 行）：

```python
if block.name == "shutdown_response" and block.input.get("approve"):
    should_exit = True
```

当队友的 LLM 调用了 `shutdown_response(approve=True)` 时，设置 `should_exit` 标志。循环在下一次迭代开始时检查此标志并退出：

```python
if should_exit:
    break
```

循环退出后，状态更新逻辑（第 216-219 行）：

```python
member = self._find_member(name)
if member:
    member["status"] = "shutdown" if should_exit else "idle"
    self._save_config()
```

注意：如果 `should_exit=True`，最终状态是 `"shutdown"` 而非 `"idle"`。这与 s09 中无条件标记为 `"idle"` 不同。

### 3.4 Lead 端查询关机状态

Lead 的 `shutdown_response` 工具（注意命名，这不是队友的工具）用于查询关机请求的状态：

```python
# TOOL_HANDLERS 中的注册（第 392 行）
"shutdown_response": lambda **kw: _check_shutdown_status(kw.get("request_id", "")),
```

```python
def _check_shutdown_status(request_id: str) -> str:
    with _tracker_lock:
        return json.dumps(shutdown_requests.get(request_id, {"error": "not found"}))
```

**完整的关机握手流程**：

```
时间线：
  t1: Lead 调用 shutdown_request(teammate="bob")
      → 生成 req_id="a1b2c3d4"
      → shutdown_requests["a1b2c3d4"] = {"target":"bob", "status":"pending"}
      → 发送 shutdown_request 消息到 bob.jsonl

  t2: Bob 的循环中 drain 收件箱，收到 shutdown_request 消息
      → 消息注入 Bob 的对话历史
      → LLM 决策：调用 shutdown_response(request_id="a1b2c3d4", approve=True)

  t3: _exec() 处理 shutdown_response 工具
      → shutdown_requests["a1b2c3d4"]["status"] = "approved"
      → 发送 shutdown_response 消息到 lead.jsonl
      → should_exit = True

  t4: Bob 的循环下一次迭代检测 should_exit=True → break
      → member["status"] = "shutdown"
      → _save_config()

  t5: Lead 调用 shutdown_response(request_id="a1b2c3d4")
      → 返回 {"target":"bob", "status":"approved"}
```

### 3.5 关机工具定义

**Lead 端的关机工具**（2 个）：

```python
# shutdown_request：发起关机
{
    "name": "shutdown_request",
    "description": "Request a teammate to shut down gracefully. Returns a request_id for tracking.",
    "input_schema": {
        "type": "object",
        "properties": {"teammate": {"type": "string"}},
        "required": ["teammate"]
    }
}

# shutdown_response：查询状态（注意不是审批！）
{
    "name": "shutdown_response",
    "description": "Check the status of a shutdown request by request_id.",
    "input_schema": {
        "type": "object",
        "properties": {"request_id": {"type": "string"}},
        "required": ["request_id"]
    }
}
```

**队友端的关机工具**（1 个）：

```python
{
    "name": "shutdown_response",
    "description": "Respond to a shutdown request. Approve to shut down, reject to keep working.",
    "input_schema": {
        "type": "object",
        "properties": {
            "request_id": {"type": "string"},
            "approve": {"type": "boolean"},
            "reason": {"type": "string"}
        },
        "required": ["request_id", "approve"]
    }
}
```

注意：`shutdown_response` 在两端有不同实现——Lead 端查询状态（`_check_shutdown_status`），队友端做决策（`_exec` 中处理 approve/reject）。

---

## 4. 计划审批协议

### 4.1 问题场景

```
场景：高风险操作
Lead -> Bob: "重构整个认证模块"
Bob [立即开始，可能破坏系统！]

更好的方式：
Lead -> Bob: "重构整个认证模块"
Bob [先写计划]
Bob -> Lead: plan_approval(plan="重构计划...")
Lead [审查计划]
Lead -> Bob: plan_approval(request_id="e5f6a1b2", approve=True, feedback="...")
Bob [收到批准通知，开始执行]
```

### 4.2 源码分析：队友提交计划

队友的 `_exec()` 方法中处理 `plan_approval` 工具（第 246-255 行）：

```python
if tool_name == "plan_approval":
    plan_text = args.get("plan", "")
    req_id = str(uuid.uuid4())[:8]              # UUID 前 8 位
    with _tracker_lock:
        plan_requests[req_id] = {
            "from": sender,                      # 提交者名称
            "plan": plan_text,                   # 计划内容
            "status": "pending"                  # 初始状态
        }
    BUS.send(
        sender, "lead", plan_text,               # 消息内容是计划文本
        "plan_approval_response",                # 消息类型
        {"request_id": req_id, "plan": plan_text} # extra 字段
    )
    return f"Plan submitted (request_id={req_id}). Waiting for lead approval."
```

**关键细节**：
1. **request_id 在队友端生成** —— 与关机协议（Lead 端生成）不同，计划的 request_id 由提交计划的队友生成。
2. **消息类型 `plan_approval_response` 的命名** —— 虽然名字含 "response"，但这是队友发给 Lead 的请求消息。命名采用"从 Lead 视角出发"的惯例：Lead 收到后需做出审批决策，视为一种"待审批的响应"。后续 Lead 审批后也复用同一类型发回通知，使该类型承担双向通信角色，命名略显反直觉。
3. **`plan_requests` 字典** —— 记录 `from`（提交者）、`plan`（计划文本）、`status`（初始 pending）。

### 4.3 源码分析：Lead 审批计划

`handle_plan_review()` 函数（第 361-372 行）处理 Lead 的审批决策：

```python
def handle_plan_review(request_id: str, approve: bool, feedback: str = "") -> str:
    with _tracker_lock:
        req = plan_requests.get(request_id)
    if not req:
        return f"Error: Unknown plan request_id '{request_id}'"
    with _tracker_lock:
        req["status"] = "approved" if approve else "rejected"
    BUS.send(
        "lead", req["from"], feedback,
        "plan_approval_response",
        {"request_id": request_id, "approve": approve, "feedback": feedback},
    )
    return f"Plan {req['status']} for '{req['from']}'"
```

**注意锁的使用模式**：这里用了两次 `with _tracker_lock`——第一次读取请求，第二次更新状态。两次加锁之间的 `if not req` 判断在锁外执行错误检查，避免在锁内返回。这个实现有一个微妙之处：两次加锁之间，理论上另一个线程可能修改了 `plan_requests`（当前代码中不太可能发生）。

### 4.4 计划审批的完整代码路径

```
时间线：
  t1: Bob 的 LLM 决定提交计划，调用 plan_approval(plan="重构认证模块：...")

  t2: _exec() 处理 plan_approval 工具
      → 生成 req_id="e5f6a1b2"
      → plan_requests["e5f6a1b2"] = {"from":"bob", "plan":"...", "status":"pending"}
      → 发送 plan_approval_response 消息到 lead.jsonl

  t3: Lead 的主循环 drain 收件箱
      → 收到 plan_approval_response 消息（含 request_id、plan）
      → LLM 决策：调用 plan_approval(request_id="e5f6a1b2", approve=True, feedback="计划可行")

  t4: handle_plan_review() 执行
      → plan_requests["e5f6a1b2"]["status"] = "approved"
      → 发送 plan_approval_response 消息到 bob.jsonl（含 approve=True, feedback）

  t5: Bob 的循环 drain 收件箱
      → 收到审批结果通知
      → LLM 看到批准，开始执行计划
```

---

## 5. 工具集成

### 5.1 Lead 端的 12 个工具

s10 在 s09 的 9 个工具基础上新增 3 个协议工具：

```python
TOOL_HANDLERS = {
    # s09 继承的 9 个工具
    "bash":              lambda **kw: _run_bash(kw["command"]),
    "read_file":         lambda **kw: _run_read(kw["path"], kw.get("limit")),
    "write_file":        lambda **kw: _run_write(kw["path"], kw["content"]),
    "edit_file":         lambda **kw: _run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "spawn_teammate":    lambda **kw: TEAM.spawn(kw["name"], kw["role"], kw["prompt"]),
    "list_teammates":    lambda **kw: TEAM.list_all(),
    "send_message":      lambda **kw: BUS.send("lead", kw["to"], kw["content"], kw.get("msg_type", "message")),
    "read_inbox":        lambda **kw: json.dumps(BUS.read_inbox("lead"), indent=2),
    "broadcast":         lambda **kw: BUS.broadcast("lead", kw["content"], TEAM.member_names()),

    # s10 新增的 3 个协议工具
    "shutdown_request":  lambda **kw: handle_shutdown_request(kw["teammate"]),
    "shutdown_response": lambda **kw: _check_shutdown_status(kw.get("request_id", "")),
    "plan_approval":     lambda **kw: handle_plan_review(kw["request_id"], kw["approve"], kw.get("feedback", "")),
}
```

### 5.2 队友端的 8 个工具

s10 在 s09 的 6 个工具基础上新增 2 个协议工具：

```python
# s09 继承的 6 个工具
bash, read_file, write_file, edit_file, send_message, read_inbox

# s10 新增的 2 个协议工具
shutdown_response    # 响应关机请求（approve/reject）
plan_approval        # 提交计划等待审批
```

### 5.3 工具命名的对称性

同名工具在两端有不同的语义，这是角色分离的设计：

| 动作 | Lead 工具 | 队友工具 |
|------|-----------|----------|
| 发起关机 | `shutdown_request(teammate)` | （被动接收消息） |
| 响应关机 | `shutdown_response(request_id)` （查询状态） | `shutdown_response(request_id, approve)` （做决策） |
| 提交计划 | （被动接收消息） | `plan_approval(plan)` |
| 审批计划 | `plan_approval(request_id, approve)` | （被动接收消息） |

同名工具在两端有不同的语义，这是需要理解的设计特点。

---

## 6. 队友循环的变化（s09 vs s10）

s10 的 `_teammate_loop` 相比 s09 有三个关键变化：

### 6.1 should_exit 标志

```python
should_exit = False
for _ in range(50):
    inbox = BUS.read_inbox(name)
    for msg in inbox:
        messages.append({"role": "user", "content": json.dumps(msg)})
    if should_exit:                        # 新增：检查退出标志
        break
    # ... LLM 调用和工具执行 ...
    for block in response.content:
        if block.type == "tool_use":
            # ...
            if block.name == "shutdown_response" and block.input.get("approve"):
                should_exit = True         # 新增：关机批准后设置标志
```

s09 的循环没有 `should_exit`，只有自然结束（50 轮上限或 `stop_reason != "tool_use"`）。s10 增加了一个主动退出条件。

### 6.2 状态区分

```python
# s09: 无条件标记为 idle
if member and member["status"] != "shutdown":
    member["status"] = "idle"

# s10: 根据 should_exit 区分
if member:
    member["status"] = "shutdown" if should_exit else "idle"
```

s10 引入了明确的 `"shutdown"` 终态，与 `"idle"` 区分。这使得 Lead 可以通过 `list_teammates()` 准确判断队友是"工作完成暂时空闲"还是"已被协议关闭"。

### 6.3 system prompt 的增强

```python
# s09
sys_prompt = f"You are '{name}', role: {role}, at {WORKDIR}. Use send_message to communicate. Complete your task."

# s10
sys_prompt = (
    f"You are '{name}', role: {role}, at {WORKDIR}. "
    f"Submit plans via plan_approval before major work. "
    f"Respond to shutdown_request with shutdown_response."
)
```

s10 的 system prompt 明确指导队友使用协议工具，让 LLM 知道应该"重大操作前提交计划"和"收到关机请求时用 shutdown_response 回复"。

---

## 7. 与 s09 的对比

| 方面 | s09 团队 | s10 协议 |
|------|----------|----------|
| 通信 | 自由消息（`message` / `broadcast`） | 结构化请求（`shutdown_request` / `shutdown_response` / `plan_approval_response`） |
| 状态跟踪 | 仅 `working` / `idle`（config.json） | 增加 `shutdown` 终态 + `shutdown_requests` / `plan_requests` 内存字典 |
| 关联机制 | 无 | `request_id`（UUID 前 8 位） |
| 协商 | 无 | LLM 自主决策（approve / reject） |
| 线程安全 | 无锁（每个队友只写自己的邮箱） | `_tracker_lock` 保护全局跟踪器 |
| 队友工具数 | 6 个 | 8 个（+shutdown_response, +plan_approval） |
| Lead 工具数 | 9 个 | 12 个（+shutdown_request, +shutdown_response, +plan_approval） |
| 退出机制 | 仅 daemon 线程被动终止 | 协议化优雅退出（should_exit 标志） |

---

## 8. 完整示例

### 8.1 优雅关机

```
s10 >> 请 alice 停止工作，项目已结束

Lead: shutdown_request(teammate="alice")
    → Shutdown request a1b2c3d4 sent to 'alice' (status: pending)

[Alice 的线程]
  BUS.read_inbox("alice") → 收到 shutdown_request
  LLM 决策："当前没有重要工作，同意关机"
  调用 shutdown_response(request_id="a1b2c3d4", approve=True)
    → shutdown_requests["a1b2c3d4"]["status"] = "approved"
    → 发送确认到 lead.jsonl → should_exit = True

[Alice 的下一次循环]
  if should_exit: break → member["status"] = "shutdown" → _save_config()

Lead: shutdown_response(request_id="a1b2c3d4")
    → {"target": "alice", "status": "approved"}
```

### 8.2 计划审批

```
s10 >> 让 bob 重构认证模块

Lead: send_message(to="bob", content="请重构认证模块")

[Bob 的线程]
  LLM："这是重大变更，应先提交计划"
  plan_approval(plan="重构认证模块：1. 新建 UserAuth 类 2. 迁移逻辑 3. 更新测试")
    → plan_requests["e5f6a1b2"] = {"from":"bob","plan":"...","status":"pending"}
    → 发送 plan_approval_response 到 lead.jsonl

[Lead 的收件箱]
  LLM 审查："计划合理，批准"
  plan_approval(request_id="e5f6a1b2", approve=True, feedback="计划详细，可以执行")
    → plan_requests["e5f6a1b2"]["status"] = "approved"
    → 发送 plan_approval_response 到 bob.jsonl

[Bob 的收件箱]
  收到批准通知 → LLM 开始执行
```

---

## 9. 常见问题

### Q1：为什么需要 request_id？

**答案**：关联请求和响应，在有多个并发请求时消除歧义。

```
无 request_id：                         有 request_id：
Lead -> Bob: "停止"                      Lead -> Bob: {req_id: "a1b2c3d4", type: "shutdown_request"}
Lead -> Bob: "继续"                      Lead -> Bob: {req_id: "e5f6a1b2", type: "shutdown_request"}
Bob: "哪个请求？"                        Bob: {req_id: "a1b2c3d4", approve: true}  ← 明确对应
```

### Q2：shutdown_requests 和 plan_requests 为什么用内存字典而非文件？

**答案**：协议状态是短暂（transient）的。一旦请求完成（approved/rejected），跟踪器中的数据就只是审计记录。进程重启后请求自然失效（队友线程也随之终止），不需要持久化。这与团队配置 `config.json`（记录 name、role、status，重启后需要恢复）形成对照。

### Q3：为什么 `shutdown_response` 在 Lead 和队友端有不同实现？

**答案**：角色分离的设计。Lead 作为协调者需要查询请求状态，队友作为执行者需要做出决策。虽然工具名相同，但在 `TOOL_HANDLERS`（Lead）和 `_exec`（队友）中注册了不同的处理逻辑——对 LLM 而言是统一接口，底层行为因角色而异。

### Q4：如何处理超时？

当前源码没有实现超时机制。可以基于 `shutdown_requests` 字典扩展：

```python
def check_timeouts(timeout: int = 60):
    """检查超时的请求（需在 handle_shutdown_request 中记录 created_at）"""
    now = time.time()
    for req_id, req in list(shutdown_requests.items()):
        if req["status"] == "pending" and now - req.get("created_at", now) > timeout:
            req["status"] = "timeout"
```

### Q5：协议可以嵌套吗？

源码中的跟踪器是扁平字典，不支持嵌套。如果需要嵌套协议（如：计划审批 -> 子任务关机请求），需扩展跟踪器结构：

```python
plan_requests["abc"] = {
    "type": "plan", "status": "pending",
    "sub_requests": ["def", "ghi"],  # 关联的子请求
}
```

### Q6：两个 Agent 同时请求对方关机，会发生死锁吗？

当前架构不会出现此场景：关机协议只能由 Lead 发起（`handle_shutdown_request` 仅在 Lead 的 `TOOL_HANDLERS` 中注册），队友之间不能互发 `shutdown_request`。

如果将协议扩展为队友间互相请求关机，可能出现死锁：双方都在等对方先同意。三种解决思路：

1. **引入优先级** —— 全局唯一 ID 较小的一方先批准对方（破坏环路等待）
2. **超时兜底** —— 为每个请求设超时，超时后自动 reject
3. **中心化仲裁** —— 所有关机请求经 Lead 审批，从架构层面消除环形依赖

---

## 实战应用：设计团队通信协议

### 场景一：优雅关机与交接

Alice 正在写一个 2000 行的文件，Lead 需要终止她的工作。中途停止会导致数据损坏。

```
Lead → Alice: shutdown_request {request_id: "sd01", reason: "需求变更，暂停前端开发"}
Alice 评估当前状态 → 完成当前段落的 write_file
Alice → Lead: shutdown_response(request_id="sd01", approve=True, reason="已保存当前进度")
Lead 收到 approved → 确认安全退出
```

关键原则：**被请求方（Alice 的 LLM）自主决定何时真正退出**，而非立即终止。

### 场景二：高风险操作审批

Bob 要执行数据库迁移，一旦出错会影响所有服务。

```
Bob → Lead: plan_approval {plan: "1. 备份 production DB → 2. staging 迁移 → 3. 回归测试 → 4. production 迁移"}
Lead 审查 → plan_approval(request_id="pl01", approve=True, feedback="增加步骤：staging 数据校验")
Bob 收到批准 → 按审批后的计划执行
```

关键原则：**执行者先承诺计划，管理者审查后才放行**，避免"先斩后奏"。

### 场景三：资源竞争仲裁

Alice 和 Bob 都需要独占访问测试数据库，同时操作会互相干扰。Lead 作为仲裁中心协调：

```
Alice → Lead: plan_approval {plan: "需要独占测试数据库 30 分钟"}
Lead 检查 → Bob 未在使用 → 批准
Lead → Bob: broadcast {content: "测试数据库已被 Alice 占用至 14:30"}
Bob 收到 → 推迟数据库操作
```

### 协议设计模板

以下代码骨架可复用于任何新的请求-响应协议：

```python
# ===== 第一步：定义全局跟踪器和锁 =====
my_requests = {}                       # {request_id: {"status": "pending", ...}}
_my_lock = threading.Lock()            # 线程安全保护

# ===== 第二步：定义工具 Schema（请求方） =====
{
    "name": "my_request",
    "description": "发起 XXX 请求",
    "input_schema": {
        "type": "object",
        "properties": {
            "target": {"type": "string"},
            "payload": {"type": "string"}
        },
        "required": ["target"]
    }
}

# ===== 第三步：定义工具 Schema（响应方） =====
{
    "name": "my_response",
    "description": "响应 XXX 请求",
    "input_schema": {
        "type": "object",
        "properties": {
            "request_id": {"type": "string"},
            "approve": {"type": "boolean"},
            "reason": {"type": "string"}
        },
        "required": ["request_id", "approve"]
    }
}

# ===== 第四步：请求处理器 =====
def handle_my_request(target: str, payload: str = "") -> str:
    req_id = str(uuid.uuid4())[:8]
    with _my_lock:
        my_requests[req_id] = {
            "target": target,
            "payload": payload,
            "status": "pending",
            "created_at": time.time()
        }
    BUS.send("lead", target, payload, "my_request", {"request_id": req_id})
    return f"Request {req_id} sent to '{target}' (pending)"

# ===== 第五步：响应处理器 =====
def handle_my_response(request_id: str, approve: bool, reason: str = "") -> str:
    with _my_lock:
        req = my_requests.get(request_id)
        if not req:
            return f"Error: Unknown request {request_id}"
        req["status"] = "approved" if approve else "rejected"
    BUS.send(req["target"], "lead", reason, "my_response",
             {"request_id": request_id, "approve": approve})
    return f"Request {request_id} {'approved' if approve else 'rejected'}"
```

---

## 10. 小结

### 10.1 核心要点

| 要点 | 说明 |
|------|------|
| **请求-响应模式** | 通用的协商机制，一个 FSM 两种应用 |
| **request_id** | UUID 前 8 位，关联请求和响应 |
| **状态跟踪器** | `shutdown_requests` / `plan_requests` 内存字典，`_tracker_lock` 保护线程安全 |
| **关机协议** | request -> response -> should_exit -> 状态迁移为 shutdown |
| **计划审批** | plan_approval 工具 -> plan_requests 记录 -> handle_plan_review 审批 |
| **should_exit 标志** | 队友循环的主动退出条件，s09 无此机制 |
| **工具命名对称** | 同名工具在 Lead/队友端有不同语义 |

### 10.2 协议模板

```python
# 1. 生成 request_id
request_id = str(uuid.uuid4())[:8]

# 2. 记录状态（加锁）
with _tracker_lock:
    tracker[request_id] = {"status": "pending", ...}

# 3. 发送请求（通过 MessageBus）
BUS.send(sender, to, content, "request_type", {"request_id": request_id})

# 4. 响应方处理请求，更新状态
with _tracker_lock:
    tracker[request_id]["status"] = "approved"  # or "rejected"

# 5. 发送响应
BUS.send(to, sender, response, "response_type", {"request_id": request_id, ...})
```

### 10.3 关键洞察

```
协议的核心价值：
1. 结构化通信 —— 标准化的请求/响应格式
2. 状态跟踪 —— 全局字典记录每个请求的生命周期
3. 双向确认 —— 请求方和响应方都明确最终结果
4. 可扩展性 —— 同一模式适用于关机、计划审批等场景

设计原则：
- 一个 FSM（pending -> approved | rejected），两种应用
- request_id 关联请求和响应，支持并发
- _tracker_lock 确保线程安全
- should_exit 标志实现优雅退出
- 队友的 LLM 自主决策 approve/reject，非硬编码
```

---

## 11. 练习

### 练习 1：优先级协议

实现带优先级的消息协议，高优先级消息优先处理。提示：在 `MessageBus` 中按优先级排序 inbox，或在 `_teammate_loop` 中优先处理高优先级消息。

### 练习 2：投票协议

实现投票机制，多个队友对决策进行投票。提示：扩展 `shutdown_requests` 的模式，增加 `vote_requests` 跟踪器，支持多响应方。

### 练习 3：协议超时

添加超时处理，自动取消长时间未响应的请求。提示：在 `handle_shutdown_request()` 中记录 `created_at`，在 Lead 的主循环中定期检查超时。

### 练习 4：协议日志

记录所有协议交互到日志文件，便于审计。提示：在 `BUS.send()` 中，当 `msg_type` 为协议类型时额外写入一个 `protocol.log` 文件。

---

## 12. 下一步

现在团队有协议了，但 Lead 仍需要手动分配任务给每个队友。能否让队友主动认领任务？

在 [s11：自主 Agent](./s11-autonomous-agents.md) 中，我们将学习：
- 空闲循环机制
- 任务自动认领
- 身份重注入
- 自主决策

---

< 上一章：[s09-Agent 团队](./s09-agent-teams.md) | [目录](./index.md) | 下一章：[s11-自主 Agent](./s11-autonomous-agents.md) >
