# s10：团队协议 —— 协作的艺术

> **核心口号**：*"队友需要共享通信规则"*

> **学习目标**：理解协商协议的必要性，掌握请求-响应模式，学会设计状态机

---

## 学习目标

完成本章后，你将能够：

1. **理解为什么需要协议** —— 自由通信的问题
2. **掌握请求-响应模式** —— 通用的协商机制
3. **实现状态机** —— pending → approved/rejected
4. **应用协议到多个场景** —— 关机、计划审批等

---

## 0. 上手演练（建议先做）

先运行团队协议示例，再阅读请求-响应状态机：

```bash
uv run python agents/s10_team_protocols.py
```

建议观察：

1. 同一 `request_id` 如何贯穿请求与响应。
2. `pending → approved/rejected` 状态迁移何时发生。
3. 协议化通信如何降低“误停机、误执行”的协作风险。

---

## 1. 问题的本质

### 1.1 自由通信的问题

在 s09 中，队友可以自由发送消息：

```
Lead → Bob: "请停止工作"
Bob [立即停止，文件可能损坏！]
```

**问题**：
1. **没有协商**：Bob 可能正在写入重要文件
2. **没有确认**：Lead 不知道 Bob 是否真的停止了
3. **没有理由**：Bob 不知道为什么要停止

### 1.2 协议化的通信

```
Lead → Bob: shutdown_request {request_id: "abc", reason: "计划变更"}
Bob 收到请求
Bob → Lead: shutdown_response {request_id: "abc", approve: true}
Bob [完成当前工作，清理，退出]
Lead [收到确认，更新状态]
```

**改进**：
1. **显式请求**：使用标准化的请求格式
2. **请求 ID**：关联请求和响应
3. **协商**：被请求者可以同意或拒绝
4. **确认**：请求者知道结果

---

## 2. 请求-响应模式

### 2.1 通用模式

```
请求者                            被请求者
  │                                  │
  │─── request {req_id, ...} ────────>│
  │                                  │
  │                    处理请求      │
  │                   做出决定       │
  │                                  │
  │<── response {req_id, ...} ───────┤
  │                                  │
  收到结果，更新状态
```

### 2.2 状态机

```
        ┌─────────────┐
        │   pending   │ ◄──── 创建请求
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

---

## 3. 关机协议

### 3.1 问题场景

```
场景 1：粗暴关机
Lead → Bob: "停止！"
Bob [立即终止]
结果：文件损坏，状态丢失

场景 2：优雅关机
Lead → Bob: "请停止工作"
Bob [完成当前文件]
Bob [保存状态]
Bob [通知完成]
Lead [更新配置]
结果：干净退出
```

### 3.2 协议设计

```python
# 全局状态跟踪
shutdown_requests = {}

# ========== 请求端 ==========
def handle_shutdown_request(teammate: str, reason: str = "") -> str:
    """发起关机请求"""
    req_id = str(uuid.uuid4())[:8]

    # 记录请求状态
    shutdown_requests[req_id] = {
        "target": teammate,
        "status": "pending",
        "reason": reason
    }

    # 发送请求
    BUS.send(
        "lead",
        teammate,
        reason or "请优雅地停止工作",
        "shutdown_request",
        {"request_id": req_id}
    )

    return f"关机请求 {req_id} 已发送给 {teammate}"

# ========== 响应端 ==========
def handle_shutdown_response(
    sender: str,
    request_id: str,
    approve: bool,
    reason: str = ""
) -> str:
    """处理关机响应"""
    # 验证请求
    if request_id not in shutdown_requests:
        return f"未知请求 ID: {request_id}"

    req = shutdown_requests[request_id]

    # 验证发送者
    if req["target"] != sender:
        return f"请求 {request_id} 不是发送给 {sender} 的"

    # 更新状态
    req["status"] = "approved" if approve else "rejected"
    req["response_reason"] = reason

    # 发送确认
    BUS.send(
        "lead",
        sender,
        f"关机请求已{'同意' if approve else '拒绝'}",
        "shutdown_ack",
        {"request_id": request_id}
    )

    return f"关机请求 {request_id}: {req['status']}"
```

### 3.3 队友侧的实现

```python
def _teammate_loop(self, name: str, role: str, prompt: str):
    messages = [{"role": "user", "content": prompt}]

    for _ in range(MAX_ROUNDS):
        # 检查邮箱
        inbox = self.bus.read_inbox(name)
        if inbox != "[]":
            msgs = json.loads(inbox)

            # 检查关机请求
            for msg in msgs:
                if msg.get("type") == "shutdown_request":
                    # LLM 决定是否同意
                    decision_msg = messages.copy()
                    decision_msg.append({
                        "role": "user",
                        "content": f"""收到关机请求：
请求 ID: {msg['request_id']}
原因: {msg.get('extra', {}).get('reason', '')}

你当前正在进行重要工作吗？如果是，可以拒绝。
请调用 shutdown_response 工具回复。"""
                    })

                    # 让 LLM 决策
                    response = client.messages.create(
                        model=MODEL,
                        messages=decision_msg,
                        tools=[{
                            "name": "shutdown_response",
                            "input_schema": {
                                "type": "object",
                                "properties": {
                                    "request_id": {"type": "string"},
                                    "approve": {"type": "boolean"},
                                    "reason": {"type": "string"}
                                },
                                "required": ["request_id", "approve"]
                            }
                        }],
                        max_tokens=1000
                    )

                    # 执行决策
                    for block in response.content:
                        if block.type == "tool_use" and block.name == "shutdown_response":
                            handle_shutdown_response(
                                name,
                                block.input["request_id"],
                                block.input["approve"],
                                block.input.get("reason", "")
                            )

                    # 如果同意，准备退出
                    if block.input.get("approve"):
                        # 完成当前工作
                        self._cleanup(name)
                        return  # 退出循环

        # 正常的 Agent 循环...
```

---

## 4. 计划审批协议

### 4.1 问题场景

```
场景：高风险操作
Lead → Bob: "重构整个认证模块"
Bob [立即开始，可能破坏系统！]

更好的方式：
Lead → Bob: "重构整个认证模块"
Bob [先写计划]
Bob → Lead: "这是我的重构计划..."
Lead [审查计划]
Lead → Bob: "计划批准，请执行"
Bob [开始执行]
```

### 4.2 协议设计

```python
plan_requests = {}

def submit_plan(
    from_member: str,
    plan: str,
    estimated_time: str = ""
) -> str:
    """提交计划"""
    req_id = str(uuid.uuid4())[:8]

    plan_requests[req_id] = {
        "from": from_member,
        "plan": plan,
        "estimated_time": estimated_time,
        "status": "pending"
    }

    # 发送给 Lead 审批
    BUS.send(
        from_member,
        "lead",
        f"计划待审批：{plan[:100]}...",
        "plan_request",
        {"request_id": req_id, "plan": plan, "estimated_time": estimated_time}
    )

    return f"计划 {req_id} 已提交审批"

def review_plan(
    request_id: str,
    approve: bool,
    feedback: str = ""
) -> str:
    """审批计划"""
    if request_id not in plan_requests:
        return f"未知请求 ID: {request_id}"

    req = plan_requests[request_id]
    req["status"] = "approved" if approve else "rejected"
    req["feedback"] = feedback

    # 通知提交者
    BUS.send(
        "lead",
        req["from"],
        f"你的计划已{'批准' if approve else '拒绝'}。{feedback}",
        "plan_approval",
        {"request_id": request_id, "approve": approve, "feedback": feedback}
    )

    return f"计划 {request_id}: {req['status']}"
```

---

## 5. 工具集成

### 5.1 关机工具

```python
TOOLS.extend([
    {
        "name": "request_shutdown",
        "description": "请求队友优雅关机",
        "input_schema": {
            "type": "object",
            "properties": {
                "teammate": {"type": "string"},
                "reason": {"type": "string"}
            },
            "required": ["teammate"]
        }
    },
    {
        "name": "shutdown_response",
        "description": "响应关机请求",
        "input_schema": {
            "type": "object",
            "properties": {
                "request_id": {"type": "string"},
                "approve": {"type": "boolean"},
                "reason": {"type": "string"}
            },
            "required": ["request_id", "approve"]
        }
    },
    {
        "name": "check_shutdown_requests",
        "description": "检查关机请求状态",
        "input_schema": {
            "type": "object",
            "properties": {
                "request_id": {"type": "string"}
            },
            "required": []
        }
    },
])

# 计划工具
TOOLS.extend([
    {
        "name": "submit_plan",
        "description": "提交计划供审批",
        "input_schema": {
            "type": "object",
            "properties": {
                "plan": {"type": "string"},
                "estimated_time": {"type": "string"}
            },
            "required": ["plan"]
        }
    },
    {
        "name": "review_plan",
        "description": "审批计划",
        "input_schema": {
            "type": "object",
            "properties": {
                "request_id": {"type": "string"},
                "approve": {"type": "boolean"},
                "feedback": {"type": "string"}
            },
            "required": ["request_id", "approve"]
        }
    },
])
```

### 5.2 注册处理器

```python
TOOL_HANDLERS.update({
    # 关机
    "request_shutdown": lambda **kw: handle_shutdown_request(kw["teammate"], kw.get("reason", "")),
    "shutdown_response": lambda **kw: handle_shutdown_response(kw["sender"], kw["request_id"], kw["approve"], kw.get("reason", "")),
    "check_shutdown_requests": lambda **kw: json.dumps(shutdown_requests, indent=2),

    # 计划
    "submit_plan": lambda **kw: submit_plan(kw["sender"], kw["plan"], kw.get("estimated_time", "")),
    "review_plan": lambda **kw: review_plan(kw["request_id"], kw["approve"], kw.get("feedback", "")),
})
```

---

## 6. 完整示例

### 6.1 优雅关机

```
Lead:
  request_shutdown(teammate="alice", reason="项目结束")
  → 关机请求 abc123 已发送给 alice

[Alice 的循环中]
  收到关机请求
  思考："我当前没有重要工作，同意关机"
  shutdown_response(request_id="abc123", approve=True, reason="好的")
  → 发送确认，清理状态，退出

Lead:
  收到确认
  更新配置：alice 状态 → shutdown
```

### 6.2 计划审批

```
Bob:
  submit_plan(plan="重构认证模块：1. 新建 UserAuth 类 2. 迁移逻辑 3. 更新测试", estimated_time="2小时")
  → 计划 def456 已提交审批

[Lead 的收件箱]
  收到计划请求
  思考："这个计划合理，批准"
  review_plan(request_id="def456", approve=True, feedback="计划详细，可以执行")

[Bob 的收件箱]
  收到批准通知
  开始执行计划
```

---

## 7. 与 s09 的对比

| 方面 | s09 团队 | s10 协议 |
|------|----------|----------|
| 通信 | 自由消息 | 结构化请求 |
| 状态 | 无 | pending/approved/rejected |
| 关联 | 无 | request_id |
| 协商 | 无 | 双向确认 |
| 适用场景 | 简单通知 | 重要决策 |

---

## 8. 常见问题

### Q1：为什么需要 request_id？

**答案**：关联请求和响应，特别是在有多个并发请求时。

```
无 request_id（混乱）：
Lead → Bob: "停止"
Lead → Bob: "继续"
Bob: "哪个请求？"

有 request_id（清晰）：
Lead → Bob: {req_id: "abc", action: "stop"}
Lead → Bob: {req_id: "def", action: "continue"}
Bob: {req_id: "abc", response: "ok"}
```

### Q2：协议可以嵌套吗？

**答案**：可以，但要小心处理状态。

```
嵌套示例：
Plan Request → Subtask 1 → Shutdown Request → Subtask 2 → Plan Response

状态跟踪需要嵌套结构：
{
  "abc": {
    "type": "plan",
    "status": "pending",
    "sub_requests": ["def", "ghi"]
  }
}
```

### Q3：如何处理超时？

```python
def check_timeouts():
    """检查超时的请求"""
    now = time.time()
    timeout = 60  # 60 秒超时

    for req_id, req in list(shutdown_requests.items()):
        if req["status"] == "pending":
            age = now - req.get("created_at", now)
            if age > timeout:
                req["status"] = "timeout"
                # 通知相关方
```

---

## 9. 小结

### 9.1 核心要点

| 要点 | 说明 |
|------|------|
| **请求-响应模式** | 通用的协商机制 |
| **状态机** | pending → approved/rejected |
| **request_id** | 关联请求和响应 |
| **双向确认** | 确保通信成功 |

### 9.2 协议模板

```python
# 1. 发起请求
request_id = generate_id()
requests[request_id] = {"status": "pending", ...}
BUS.send(sender, to, content, "request_type", {"request_id": request_id})

# 2. 处理请求
request = requests[request_id]
# 做出决定...

# 3. 响应请求
request["status"] = "approved"  # or "rejected"
BUS.send(to, sender, response, "response_type", {"request_id": request_id})
```

### 9.3 关键洞察

```
协议的核心价值：
1. 结构化通信：统一的消息格式
2. 状态跟踪：知道每个请求的状态
3. 双向确认：双方都明确结果
4. 可扩展性：同一模式适用于多种场景

设计原则：
- 一个 FSM，多种应用
- request_id 关联请求和响应
- pending → approved | rejected 状态转换
- 超时处理
```

---

## 10. 练习

### 练习 1：优先级协议

实现带优先级的消息协议，高优先级消息优先处理。

### 练习 2：投票协议

实现投票机制，多个队友对决策进行投票。

### 练习 3：协议超时

添加超时处理，自动取消长时间未响应的请求。

### 练习 4：协议日志

记录所有协议交互到日志文件，便于审计。

---

## 11. 下一步

现在团队有协议了，但 Lead 仍需要手动分配任务给每个队友。能否让队友主动认领任务？

在 [s11：自主 Agent](./s11-autonomous-agents.md) 中，我们将学习：
- 空闲循环机制
- 任务自动认领
- 身份重注入
- 自主决策

---

< 上一章：[s09-Agent 团队](./s09-agent-teams.md) | [目录](./index.md) | 下一章：[s11-自主 Agent](./s11-autonomous-agents.md) >
