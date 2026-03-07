# s11：自主 Agent —— 自我驱动的团队

> **核心口号**：*"队友看板并自己认领任务"*

> **学习目标**：理解自主性设计，掌握空闲循环机制，学会任务自动认领

---

## 学习目标

完成本章后，你将能够：

1. **理解自主性的价值** —— 从被动执行到主动认领
2. **掌握空闲循环** —— WORK 和 IDLE 状态切换
3. **实现任务扫描** —— 自动发现可认领任务
4. **设计身份重注入** —— 压缩后恢复身份

---

## 0. 上手演练（建议先做）

先运行自主 Agent 示例，再阅读 WORK/IDLE 循环：

```bash
uv run python agents/s11_autonomous_agents.py
```

建议观察：

1. 队友何时从 WORK 切换到 IDLE。
2. 空闲状态下如何扫描任务并自动认领。
3. 压缩后身份信息如何被重注入并保持角色稳定。

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

问题：Lead 需要手动分配每个任务
```

### 1.2 自主执行

```
Lead: 创建任务板
Lead: spawn(name="alice", role="coder")
Lead: spawn(name="bob", role="tester")

[Alice 的空闲循环]
  扫描任务板 → 发现 "实现用户登录" (待认领)
  认领任务 → 标记为 in_progress
  执行任务 → 完成

[Bob 的空闲循环]
  扫描任务板 → 发现 "编写测试" (待认领)
  认领任务 → 标记为 in_progress
  执行任务 → 完成

[完成后的 Alice]
  扫描任务板 → 发现 "修复 bug" (待认领)
  自动认领并执行
...
```

---

## 2. WORK/IDLE 循环

### 2.1 状态机

```
         创建
          │
          ▼
    ┌──────────┐
    │  WORK    │ ◄────── 收到消息 / 认领任务
    │  (执行)  │
    └────┬─────┘
         │
         │ stop_reason != "tool_use"
         │ 或调用 idle 工具
         ▼
    ┌──────────┐
    │  IDLE    │ ◄──┐
    │  (等待)  │    │ 超时
    └────┬─────┘    │ (60s)
         │          │
    ┌────┴────────┴────┐
    │                 │
 收到消息          无消息
 或发现任务        60 秒
    │                 │
    ▼                 ▼
  WORK             SHUTDOWN
```

### 2.2 伪代码

```python
def _teammate_loop(self, name: str, role: str, prompt: str):
    while True:
        # ========== WORK 阶段 ==========
        messages = self._build_messages(name, role, prompt)

        for _ in range(MAX_WORK_ROUNDS):
            response = client.messages.create(...)

            if response.stop_reason != "tool_use":
                break  # LLM 停止调用工具，进入 IDLE

            # 执行工具...
            if called_idle:
                break  # 主动请求进入 IDLE

        # ========== IDLE 阶段 ==========
        self._set_status(name, "idle")

        resume = self._idle_poll(name, messages)

        if not resume:
            self._set_status(name, "shutdown")
            return  # 超时退出

        self._set_status(name, "working")
        # 回到 WORK 阶段
```

---

## 3. 空闲轮询

### 3.1 轮询逻辑

```python
POLL_INTERVAL = 5  # 每 5 秒轮询一次
IDLE_TIMEOUT = 60   # 60 秒无工作则退出

def _idle_poll(self, name: str, messages: list) -> bool:
    """空闲阶段轮询，返回是否应该继续"""
    for _ in range(IDLE_TIMEOUT // POLL_INTERVAL):  # 最多 12 次
        time.sleep(POLL_INTERVAL)

        # 1. 检查收件箱
        inbox = BUS.read_inbox(name)
        if inbox != "[]":
            # 有新消息，恢复工作
            messages.append({
                "role": "user",
                "content": f"<inbox>\n{inbox}\n</inbox>"
            })
            return True

        # 2. 扫描任务板
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            # 有可认领的任务
            task = unclaimed[0]
            claim_task(task["id"], name)
            messages.append({
                "role": "user",
                "content": f"<auto-claimed>任务 #{task['id']}: {task['subject']}</auto-claimed>"
            })
            return True

        # 3. 超时检查
        # (继续下一次循环)

    # 超时，应该退出
    return False
```

### 3.2 任务扫描

```python
def scan_unclaimed_tasks() -> list:
    """扫描待认领的任务"""
    unclaimed = []

    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())

        # 条件：
        # 1. 状态是 pending
        # 2. 没有所有者
        # 3. 没有阻塞依赖
        if (task.get("status") == "pending"
                and not task.get("owner")
                and not task.get("blockedBy")):
            unclaimed.append(task)

    return unclaimed
```

### 3.3 任务认领

```python
def claim_task(task_id: int, claimant: str) -> str:
    """认领任务"""
    task = TASKS._load(task_id)

    # 检查任务状态
    if task.get("owner"):
        return f"任务已被 {task['owner']} 认领"

    # 更新任务
    TASKS.update(task_id, status="in_progress")

    # 设置所有者
    task["owner"] = claimant
    TASKS._save(task)

    return f"任务 #{task_id} 被 {claimant} 认领"
```

---

## 4. 身份重注入

### 4.1 问题

```
Alice 的上下文：
[系统提示：你是 Alice，角色是 coder]
[用户：实现用户登录]
[工具调用：write_file]
[工具结果：...]
...
[上下文压缩发生！]
[压缩摘要：Alice 完成了登录功能]

Alice 现在不知道自己是谁了！
```

### 4.2 解决方案

```python
def _build_messages(self, name: str, role: str, prompt: str) -> list:
    """构建消息列表，必要时注入身份"""
    messages = [{"role": "user", "content": prompt}]

    # 如果上下文太短（可能被压缩了），注入身份
    if len(messages) <= 3:
        identity = {
            "role": "user",
            "content": f"<identity>\n"
                      f"名字: {name}\n"
                      f"角色: {role}\n"
                      f"任务: 继续你的工作\n"
                      f"</identity>"
        }
        messages.insert(0, identity)

        # 添加确认
        messages.append({
            "role": "assistant",
            "content": f"我是 {name}，继续工作。"
        })

    return messages
```

---

## 5. 完整流程

### 5.1 初始化

```
Lead:
  spawn(name="alice", role="coder", prompt="负责后端 API 开发")
  spawn(name="bob", role="tester", prompt="负责测试和质量保证")

  # 创建任务
  task_create(subject="实现用户 API")
  task_create(subject="实现认证 API")
  task_create(subject="编写 API 测试")
```

### 5.2 自主认领

```
[Alice 的循环 - WORK 阶段]
  启动，进入 WORK
  LLM：我需要查看任务板
        task_list()
  结果：[ ] #1: 实现用户 API
       [ ] #2: 实现认证 API
       [ ] #3: 编写 API 测试

  LLM：我认领任务 #1
        task_update(task_id=1, status="in_progress")
        [任务认领，owner 设置为 alice]

  LLM：开始实现用户 API
        write_file("api/users.py", ...)
        [工作...]

  LLM：任务完成
        task_update(task_id=1, status="completed")

  LLM：没有更多工具调用
        stop_reason = "end_turn"

[Alice 的循环 - IDLE 阶段]
  状态变为 idle
  轮询收件箱 → 空
  轮询任务板 → 发现任务 #2 待认领
  认领任务 #2
  返回 WORK 阶段

[Alice 的循环 - WORK 阶段]
  LLM：我认领了任务 #2，开始实现认证 API
  ...
```

### 5.3 并行执行

```
同时发生：
[Alice]                    [Bob]
├── WORK: 实现 API #1    ├── WORK: 编写测试 #3
├── IDLE                 ├── IDLE
├── WORK: 实现 API #2    ├── WORK: 运行测试
├── IDLE                 ├── IDLE
└── WORK: ...            └── WORK: ...

两个队友自主工作，Lead 无需干预
```

---

## 6. 工具集成

### 6.1 新增工具

```python
TOOLS.extend([
    {
        "name": "idle",
        "description": "主动进入空闲状态，等待新任务或消息",
        "input_schema": {
            "type": "object",
            "properties": {},
            "required": []
        }
    },
    {
        "name": "claim_task",
        "description": "认领任务（通常自动执行）",
        "input_schema": {
            "type": "object",
            "properties": {
                "task_id": {"type": "integer"}
            },
            "required": ["task_id"]
        }
    },
])
```

### 6.2 注册处理器

```python
TOOL_HANDLERS.update({
    "idle": lambda **kw: "_enter_idle_",
    "claim_task": lambda **kw: claim_task(kw["task_id"], kw.get("_claimer", "unknown")),
})
```

---

## 7. 与 s10 的对比

| 方面 | s10 协议 | s11 自主 |
|------|----------|---------|
| 任务分配 | Lead 手动分配 | 队友自动认领 |
| 触发方式 | 收到消息 | 收到消息 + 扫描任务板 |
| 状态 | working/idle/shutdown | + WORK/IDLE 阶段 |
| 超时 | 无 | 60 秒无工作自动退出 |
| 身份 | 系统提示 | + 压缩后重注入 |

---

## 8. 常见问题

### Q1：多个队友同时认领同一任务怎么办？

```python
def claim_task(task_id: int, claimant: str) -> str:
    task = TASKS._load(task_id)

    # 检查是否已被认领
    if task.get("owner"):
        if task["owner"] == claimant:
            return f"你已认领此任务"
        else:
            return f"任务已被 {task['owner']} 认领"

    # 使用文件锁防止并发
    with file_lock(task_id):
        task = TASKS._load(task_id)  # 重新检查
        if task.get("owner"):
            return f"任务被抢先认领"

        task["owner"] = claimant
        TASKS._save(task)

    return f"任务被 {claimant} 认领"
```

### Q2：如何控制队友的认领偏好？

```python
# 在队友配置中添加偏好
member = {
    "name": "alice",
    "role": "coder",
    "preferences": {
        "task_types": ["api", "backend"],
        "avoid": ["testing", "docs"]
    }
}

# 扫描时考虑偏好
def scan_unclaimed_tasks(name: str) -> list:
    preferences = get_preferences(name)
    # 过滤符合偏好的任务
    return [t for t in all_tasks if matches(t, preferences)]
```

### Q3：队友可以拒绝认领任务吗？

**答案**：可以。队友在自动认领前可以先检查任务详情，决定是否认领。

```python
def _idle_poll(self, name: str, messages: list) -> bool:
    unclaimed = scan_unclaimed_tasks()

    if unclaimed:
        task = unclaimed[0]

        # 让 LLM 决定是否认领
        decision = ask_llm(
            f"发现任务：{task['subject']}\n"
            f"描述：{task['description']}\n"
            f"要认领这个任务吗？"
        )

        if decision == "yes":
            claim_task(task["id"], name)
            return True
        else:
            # 跳过这个任务
            skip_task(name, task["id"])
            # 继续轮询
```

---

## 9. 小结

### 9.1 核心要点

| 要点 | 说明 |
|------|------|
| **自主性** | 队友主动认领任务，无需分配 |
| **WORK/IDLE 循环** | 执行和等待两阶段 |
| **空闲轮询** | 定期检查收件箱和任务板 |
| **身份重注入** | 压缩后恢复身份 |

### 9.2 循环模板

```python
def _teammate_loop(self, name, role, prompt):
    while True:
        # WORK 阶段：执行任务
        for _ in range(MAX_ROUNDS):
            response = client.messages.create(...)
            if stop_reason != "tool_use":
                break
            execute_tools(...)

        # IDLE 阶段：等待新工作
        set_status("idle")
        resume = idle_poll()
        if not resume:
            return  # 超时退出
        set_status("working")
```

### 9.3 关键洞察

```
自主性的核心价值：
1. 自我驱动：主动寻找工作
2. 并行执行：多个队友同时工作
3. 负载均衡：任务自动分配
4. 弹性伸缩：空闲队友自动退出

设计原则：
- WORK 阶段：专注执行
- IDLE 阶段：定期轮询
- 超时退出：避免资源浪费
- 身份恢复：压缩后重注入
```

---

## 10. 练习

### 练习 1：任务优先级

修改扫描逻辑，优先认领高优先级任务。

### 练习 2：协作认领

实现协作机制，队友可以请求帮助，其他队友可以响应。

### 练习 3：任务移交

实现任务移交功能，队友可以将任务移交给更适合的队友。

### 练习 4：动态超时

根据任务量动态调整超时时间。

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
