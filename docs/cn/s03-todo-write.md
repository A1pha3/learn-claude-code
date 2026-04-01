# s03：任务规划 -- 让 Agent 不再迷失

> **核心口号**：*"写下来的计划才是计划，否则只是意图"*

> **学习目标**：理解规划的重要性，实现任务清单系统，掌握状态机设计

---

## 学习目标

完成本章后，你将能够：

1. **理解为什么 Agent 需要规划** -- 避免"迷失方向"的问题
2. **实现 TodoManager** -- 一个带严格校验的任务状态管理器
3. **掌握提醒机制** -- 当 Agent 偏离时自动注入提醒
4. **理解状态机设计** -- pending / in_progress / completed 三态与单一焦点约束
5. **掌握上限防护** -- 20 条上限、状态枚举校验等防御性编程技巧

### 学习分层指南

| 层级 | 掌握标准 | 对应内容 | 建议用时 |
|------|----------|----------|----------|
| L1 基础 | 能说明为什么 Agent 需要规划，能运行示例并解释 todo 列表的渲染输出 | 1.问题的本质、2.TodoManager 设计、5.完整执行流程示例 | 30 分钟 |
| L2 进阶 | 能独立实现 TodoManager 的校验逻辑，能解释提醒注入到 results 头部的精确机制 | 3.集成到 Agent、4.提醒机制详解、6.架构分析 | 45 分钟 |
| L3 专家 | 能根据项目需求扩展状态机（优先级、依赖、子任务），能设计自适应提醒策略 | 9.最佳实践、11.练习 | 60 分钟 |

---

## 0. 上手演练（建议先做）

先运行 TodoWrite 示例，再看状态机细节：

```bash
uv run python agents/s03_todo_write.py
```

建议观察：

1. 任务列表在多轮对话中如何持续可见。
2. 为什么系统强制"同一时刻只有一个 in_progress"。
3. 提醒机制触发后，Agent 的行为如何被拉回主线。
4. 尝试提交超过 20 条 todo，观察报错行为。
5. 尝试提交多个 in_progress 状态的任务，观察报错行为。

---

## 1. 问题的本质

> **概念速查：TodoManager**
>
> 一个带严格校验的任务状态管理器，维护一个字典列表 `self.items`。每个条目包含 `id`、`text`、`status` 三个字段。核心约束：同时只能有一个 `in_progress` 状态的任务（单一焦点原则），条目上限 20 条。渲染后以 `[ ]` / `[>]` / `[x]` 符号展示在每轮上下文中，防止 Agent 在多步任务中迷失方向。

> **概念速查：提醒机制（Reminder）**
>
> 当 Agent 连续 3 轮未调用 todo 工具时，系统自动在 `results` 列表头部注入 `<reminder>Update your todos.</reminder>`。提醒不重置计数器，只有实际调用 todo 工具才会重置。设计意图：以最小干预纠正 Agent 的偏离行为。

### 1.1 没有规划的 Agent 会怎样

想象一个 10 步的代码重构任务：

```
用户：重构 auth.py：
1. 添加类型注解
2. 添加文档字符串
3. 重命名函数 login_to_system 为 login
4. 提取验证逻辑到单独函数
5. 添加单元测试
6. 更新导入
7. 运行测试
8. 修复测试失败
9. 更新文档
10. 提交更改
```

**没有规划的 Agent 可能会**：

```
轮次 1：添加类型注解（步骤 1）
轮次 2：添加文档字符串（步骤 2）
轮次 3：运行测试（跳到步骤 7）
轮次 4：发现没有测试，决定写测试（步骤 5）
轮次 5：重命名函数（回到步骤 3）
轮次 6：又添加了一遍类型注解（重复步骤 1）
...
```

**问题**：Agent 忘记了完整计划，开始随机游走。

### 1.2 为什么会迷失

这与人脑的**工作记忆限制**类似：

```
+-------------------------------------+
|         LLM 的上下文窗口            |
|                                     |
|  [系统提示]                         |
|  [用户提问]                         |
|  [工具调用 1: read_file]            |
|  [工具结果 1: ...5000 tokens...]   | <-- 大量输出
|  [工具调用 2: read_file]            |
|  [工具结果 2: ...5000 tokens...]   | <-- 又大量输出
|  [工具调用 3: run_bash]             |
|  [工具结果 3: ...3000 tokens...]   |
|                                     |
|  === 系统提示被淹没 ===             |
|  === 原始计划被遗忘 ===             |
+-------------------------------------+
```

随着对话进行，工具结果堆积，早期的重要内容（如系统提示和用户指令）被新内容"挤"出注意力。

### 1.3 解决方案：显式规划

```
+-------------------------------------+
|         TodoManager 状态            |
|                                     |
|  [ ] #1: 添加类型注解               |
|  [>] #2: 添加文档字符串 <-- 正在做   |
|  [ ] #3: 重命名函数                 |
|  [ ] #4: 提取验证逻辑               |
|  [ ] #5: 添加单元测试               |
|  [x] #6: 更新导入 <-- 已完成         |
|  [ ] #7: 运行测试                   |
|  [ ] #8: 修复测试失败               |
|  [ ] #9: 更新文档                   |
|  [ ] #10: 提交更改                  |
|                                     |
|  (1/10 completed)                   |
|                                     |
|  每轮都可见，不会被淹没              |
+-------------------------------------+
```

**关键点**：Todo 状态通过工具结果持续注入到上下文中，确保 Agent 始终能看到计划。

**为什么不把计划放在系统提示中？** 系统提示是静态的，一旦设定就不会变化。而 Todo 状态需要随着任务推进动态更新（pending -> in_progress -> completed）。将其作为工具结果注入，使得每次调用 todo 工具后，新的状态自然成为上下文的一部分。这是"动态状态"与"静态指令"在架构上的根本区别。

---

## 2. TodoManager 设计

### 2.1 数据结构

源码中的 TodoManager 管理一个字典列表，每个条目包含三个必填字段：

```python
# 源码中的数据模型（简化）
{"id": "1", "text": "添加类型注解", "status": "pending"}
{"id": "2", "text": "添加文档字符串", "status": "in_progress"}
{"id": "3", "text": "重命名函数", "status": "completed"}
```

三个字段的含义：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 唯一标识，若未提供则自动按序号生成 |
| `text` | string | 任务描述，不允许为空 |
| `status` | string | 任务状态，必须为三值之一 |

### 2.2 状态定义

源码第 66 行定义了三个合法状态值：

| 状态 | 含义 | 渲染符号 | 约束 |
|------|------|----------|------|
| `pending` | 待办 | `[ ]` | 可以有多个 |
| `in_progress` | 进行中 | `[>]` | **同时只能有一个** |
| `completed` | 已完成 | `[x]` | 可以有多个 |

> **注意**：源码使用 `completed` 而非 `done`。这一点在工具的 JSON Schema 的 `enum` 定义（源码第 158 行）中也得到了确认：`["pending", "in_progress", "completed"]`。

**为什么只能有一个 in_progress？**

这迫使 Agent **聚焦单一任务**，避免同时做多件事导致混乱。从 LLM 注意力机制的角度看，多个 in_progress 会分散模型的"注意力预算"，导致每个任务都做得不完整。单一焦点约束将 Agent 的行为从"并行尝试"强制转为"串行完成"。源码第 68-72 行实现了这一约束：

```python
if status == "in_progress":
    in_progress_count += 1
# ...
if in_progress_count > 1:
    raise ValueError("Only one task can be in_progress at a time")
```

```
错误 -- 多个 in_progress（混乱）：
[>] 任务 A
[>] 任务 B  <-- 同时做两件事？
[>] 任务 C
[ ] 任务 D

正确 -- 单一 in_progress（清晰）：
[x] 任务 A
[>] 任务 B  <-- 专注当前任务
[ ] 任务 C
[ ] 任务 D
```

### 2.3 20 条上限

源码第 56-57 行设置了硬性上限：

```python
def update(self, items: list) -> str:
    if len(items) > 20:
        raise ValueError("Max 20 todos allowed")
```

**为什么是 20？**

- 每条 todo 渲染后约占 1 行（约 20-50 tokens），20 条约 400-1000 tokens
- 这个数量级在上下文中既足够表达复杂计划，又不会因过长而被注意力机制忽略
- 超过 20 条意味着任务拆分粒度过细，应考虑分层或使用子 Agent（见 s04）

### 2.4 校验流程

`update()` 方法（源码第 55-74 行）对传入的 items 列表执行严格校验：

```
items 列表进入
    |
    v
[1] 数量检查：len(items) > 20 ? --> 抛出 ValueError
    |
    v
[2] 逐条校验：
    |-- text 为空 ? --> 抛出 ValueError("text required")
    |-- status 不在 (pending, in_progress, completed) ? --> 抛出 ValueError("invalid status")
    |-- status == in_progress ? --> 计数器 +1
    |
    v
[3] 单焦点检查：in_progress_count > 1 ? --> 抛出 ValueError
    |
    v
[4] 通过校验，替换 self.items
    |
    v
[5] 调用 render() 返回渲染结果
```

**设计要点**：

- 校验失败时**整个更新被拒绝**，`self.items` 保持不变（"原子性"语义）
- `id` 字段是可选的，若未提供则自动按序号（`str(i + 1)`）生成
- `status` 字段也是可选的，默认为 `"pending"`（源码第 62 行 `item.get("status", "pending")`）
- 但在工具的 JSON Schema 中，`id`、`text`、`status` 均被标记为 `required`（源码第 158 行）

### 2.5 render() 渲染方法

源码第 76-85 行实现了渲染逻辑：

```python
def render(self) -> str:
    if not self.items:
        return "No todos."
    lines = []
    for item in self.items:
        marker = {"pending": "[ ]", "in_progress": "[>]", "completed": "[x]"}[item["status"]]
        lines.append(f"{marker} #{item['id']}: {item['text']}")
    done = sum(1 for t in self.items if t["status"] == "completed")
    lines.append(f"\n({done}/{len(self.items)} completed)")
    return "\n".join(lines)
```

渲染输出示例：

```
[ ] #1: 添加类型注解
[>] #2: 添加文档字符串
[ ] #3: 重命名函数

(0/3 completed)
```

注意格式是 `marker #id: text`（井号在 id 前面），末尾附有完成进度统计。

### 2.6 设计决策记录

| 决策 | 选择 | 备选方案 | 理由 |
|------|------|----------|------|
| 存储方式 | 内存列表 `self.items = []` | SQLite / JSON 文件 | TodoManager 是短期记忆，无需持久化；列表渲染到消息中更直接 |
| 状态数量 | 3 个（pending / in_progress / completed） | 5 个（+ blocked / cancelled） | 3 态足以覆盖核心流程，更少的状态意味着更少的 LLM 犯错空间 |
| 焦点约束 | 同一时刻只能有 1 个 in_progress | 允许多个 in_progress | 强制聚焦单一任务，防止 Agent 在任务间跳跃导致混乱 |
| 条目上限 | 20 条 | 无上限 / 50 条 | 20 条约 400-1000 tokens，既够用又不至于淹没上下文；超过 20 说明应拆分子 Agent |
| 提醒阈值 | 3 轮未调用 todo | 1 轮 / 5 轮 | 3 轮平衡了"不打断正常工作流"和"及时纠正偏离"的需求 |
| 提醒位置 | `results.insert(0, ...)` 插入到 results 头部 | 单独追加一条 user 消息 | 保持 messages 结构规整（每轮一条 assistant + 一条 user），且确保 LLM 优先看到提醒 |
| 校验策略 | 原子性：校验失败则整个更新被拒绝 | 逐条校验、跳过无效条目 | 原子性保证状态一致性，避免部分更新导致 todo 列表进入不一致状态 |
| 渲染格式 | `marker #id: text` + 末尾进度统计 | 仅列表，无统计 | 进度统计（如 `1/3 completed`）帮助 LLM 感知整体进度，减少重复已完成任务的概率 |

---

## 3. 集成到 Agent

### 3.1 添加 todo 工具

源码第 148-159 行定义了 todo 工具的 JSON Schema：

```python
{"name": "todo",
 "description": "Update task list. Track progress on multi-step tasks.",
 "input_schema": {
     "type": "object",
     "properties": {
         "items": {
             "type": "array",
             "items": {
                 "type": "object",
                 "properties": {
                     "id":     {"type": "string"},
                     "text":   {"type": "string"},
                     "status": {"type": "string",
                                "enum": ["pending", "in_progress", "completed"]}
                 },
                 "required": ["id", "text", "status"]
             }
         }
     },
     "required": ["items"]
 }}
```

关键设计：`status` 的 `enum` 约束让 LLM 在生成参数时就受到限制，只能选择三个合法值之一。

### 3.2 注册处理器

源码第 88 行创建全局实例，第 145 行注册处理器：

```python
TODO = TodoManager()

TOOL_HANDLERS = {
    # ... 其他工具 ...
    "todo": lambda **kw: TODO.update(kw["items"]),
}
```

`TODO` 是一个全局单例。这意味着在同一个会话中，todo 状态在多轮对话之间持续存在。

### 3.3 系统提示中的引导

源码第 45-47 行的系统提示明确引导 LLM 使用 todo：

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Use the todo tool to plan multi-step tasks. Mark in_progress before starting, completed when done.
Prefer tools over prose."""
```

注意这里明确提到了状态转换规则："Mark in_progress before starting, completed when done"，即：

```
pending --> in_progress --> completed
```

---

## 4. 提醒机制详解

### 4.1 为什么需要提醒

即使有 todo 工具，Agent 也可能"忘记"更新它：

```
轮次 1：创建 todo（in_progress = 任务 A）
轮次 2：完成任务 A，直接开始做任务 B <-- 忘记更新 todo
轮次 3：继续任务 B                   <-- rounds_since_todo = 1
轮次 4：继续任务 B                   <-- rounds_since_todo = 2
轮次 5：继续任务 B                   <-- rounds_since_todo = 3, 触发提醒!
```

### 4.2 提醒注入的精确位置

源码第 163-190 行实现了完整的 agent_loop，提醒注入的关键逻辑在第 188-189 行：

```python
def agent_loop(messages: list):
    rounds_since_todo = 0
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return
        results = []
        used_todo = False
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                # ... 执行工具，追加 tool_result 到 results ...
                if block.name == "todo":
                    used_todo = True
        rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
        if rounds_since_todo >= 3:
            results.insert(0, {"type": "text", "text": "<reminder>Update your todos.</reminder>"})
        messages.append({"role": "user", "content": results})
```

**提醒注入的精确机制**：

1. 提醒**不是**插入到 `messages` 列表中，而是插入到本轮的 `results` 列表的**头部**（`results.insert(0, ...)`）
2. 整个 `results` 列表随后作为一条 `role: "user"` 消息追加到 `messages` 中
3. 提醒内容是 `<reminder>Update your todos.</reminder>`，使用类 XML 标签包裹
4. 提醒**不会重置** `rounds_since_todo` 计数器。只有实际调用 todo 工具才会重置

这意味着如果 Agent 收到提醒后仍然不更新 todo，下一轮又会收到提醒（因为计数器会继续增长到 4、5...）。

### 4.3 计数器状态转换

```
                    调用 todo 工具
                 +-------------------+
                 |                   |
                 v                   |
rounds_since_todo = 0 ----> 1 ----> 2 ----> 3 ----> 4 ...
                                           |          |
                                           v          v
                                      注入提醒    注入提醒
                                    （不重置计数器）（不重置计数器）
```

### 4.4 提醒频率的权衡

| 轮次阈值 | 优点 | 缺点 |
|----------|------|------|
| 1-2 | 从不偏离 | 提醒过于频繁，可能打断正常工作流 |
| 3（源码选择） | 平衡 | 可能有些延迟 |
| 5+ | 不打扰 | 可能严重偏离 |

---

## 5. 完整执行流程示例

### 5.1 用户请求

```
重构 auth.py：添加类型注解、文档字符串、重命名 login_to_system 为 login
```

### 5.2 Agent 执行流程

```
轮次 1：
  用户：重构 auth.py...
  LLM：我需要先规划任务
        tool_use: todo(items=[
          {id: "1", text: "添加类型注解", status: "pending"},
          {id: "2", text: "添加文档字符串", status: "pending"},
          {id: "3", text: "重命名函数", status: "pending"}
        ])
  工具结果：
    [ ] #1: 添加类型注解
    [ ] #2: 添加文档字符串
    [ ] #3: 重命名函数

    (0/3 completed)
  rounds_since_todo = 0（调用了 todo）

轮次 2：
  LLM：现在开始第一个任务
        tool_use: todo(items=[...1: "in_progress"...])
        tool_use: read_file(path="auth.py")
  工具结果：
    [>] #1: 添加类型注解
    [ ] #2: 添加文档字符串
    [ ] #3: 重命名函数

    (0/3 completed)
    [文件内容...]
  rounds_since_todo = 0（调用了 todo）

轮次 3：
  LLM：添加类型注解
        tool_use: write_file(...)
  rounds_since_todo = 1（未调用 todo）

轮次 4：
  LLM：完成第一个任务，开始第二个
        tool_use: todo(items=[...1: "completed", 2: "in_progress"...])
  工具结果：
    [x] #1: 添加类型注解
    [>] #2: 添加文档字符串
    [ ] #3: 重命名函数

    (1/3 completed)
  rounds_since_todo = 0（调用了 todo）

轮次 5：
  LLM：添加文档字符串
        tool_use: edit_file(...)
  rounds_since_todo = 1

轮次 6：
  LLM：完成第二个任务，开始第三个
        tool_use: todo(items=[...2: "completed", 3: "in_progress"...])
  rounds_since_todo = 0

轮次 7：
  LLM：重命名函数
        tool_use: edit_file(...)
  rounds_since_todo = 1

轮次 8：
  LLM：完成所有任务
        tool_use: todo(items=[...all: "completed"...])
  工具结果：
    [x] #1: 添加类型注解
    [x] #2: 添加文档字符串
    [x] #3: 重命名函数

    (3/3 completed)
  rounds_since_todo = 0

轮次 9：
  LLM：重构已完成！
  stop_reason = "end_turn"，循环结束
```

---

## 6. 架构分析

### 6.1 数据流

```
+----------+      +-------+      +---------+
|   User   | ---> |  LLM  | ---> | Tools   |
|  prompt  |      |       |      | + todo  |
+----------+      +---+---+      +----+----+
                       ^               |
                       |   tool_result |
                       +---------------+
                             |
                 +-----------+-----------+
                 | TodoManager state     |
                 | [ ] #1: task A        |
                 | [>] #2: task B        |
                 | [x] #3: task C        |
                 |                       |
                 | (1/3 completed)       |
                 +-----------------------+
                             |
                 if rounds_since_todo >= 3:
                   results.insert(0, <reminder>)
```

### 6.2 工具集对比

| 工具名 | 功能 | 参数 |
|--------|------|------|
| `bash` | 执行 shell 命令 | `command` |
| `read_file` | 读取文件 | `path`, `limit`(可选) |
| `write_file` | 写入文件 | `path`, `content` |
| `edit_file` | 替换文件文本 | `path`, `old_text`, `new_text` |
| `todo` | 更新任务列表 | `items`（数组，每项含 `id`, `text`, `status`） |

与 s02 相比，s03 新增了 `todo` 工具，其余四个工具实现完全相同。

### 6.3 安全防护

源码中包含多层安全防护：

| 防护层 | 位置 | 机制 |
|--------|------|------|
| 路径穿越 | `safe_path()` | `resolve()` + `is_relative_to(WORKDIR)` 检查 |
| 危险命令 | `run_bash()` | 黑名单拦截 `rm -rf /`, `sudo`, `shutdown` 等 |
| 超时保护 | `run_bash()` | `subprocess.TimeoutExpired`，120 秒超时 |
| 输出截断 | `run_bash()`, `run_read()` | 限制 50000 字符 |
| Todo 数量 | `TodoManager.update()` | 上限 20 条 |
| Todo 状态 | `TodoManager.update()` | 枚举校验 pending/in_progress/completed |
| 单焦点约束 | `TodoManager.update()` | in_progress 只允许一个 |

---

## 7. 与 s02 的对比

| 方面 | s02 | s03 | 为什么这样演进 |
|------|-----|-----|----------------|
| 工具数量 | 4 | 5 (+todo) | 多步任务需要显式规划，否则 Agent 在长对话中随机游走 |
| 规划能力 | 无 | TodoManager（全局单例） | 工具结果是临时性的，无法持续展示计划；TodoManager 通过渲染注入提供持久可见的任务列表 |
| 状态管理 | 无 | pending / in_progress / completed | LLM 的"工作记忆"有限，需要外部状态机替代记忆 |
| 提醒机制 | 无 | 3 轮后自动注入 reminder 到 results | 即使有 todo 工具，Agent 也可能忘记更新；提醒是"自动化"层面的保障 |
| 循环结构 | `while stop_reason == "tool_use"` | 相同 + `rounds_since_todo` 计数器 | 基础循环不变，仅在循环内增加偏离检测逻辑 |
| 校验能力 | 基础错误处理 | TodoManager 内建多层校验 | 状态机如果不加约束，LLM 可能产生非法状态（如多个 in_progress），破坏聚焦原则 |

---

## 8. 常见问题

### Q1：为什么不用数据库？

```python
# 为什么用内存列表而不是数据库？
self.items = []  # 简单列表
```

**答案**：
1. TodoManager 是**短期记忆**，不需要跨会话持久化
2. 列表更简单，序列化到消息中很方便
3. s07 会引入持久化的任务系统

### Q2：可以跳过规划直接执行吗？

**答案**：可以，但不推荐。对于简单任务（如"读取文件"），直接执行更高效。但对于复杂任务（3+ 步骤），规划能显著提高成功率。系统提示中的 `Prefer tools over prose` 引导 LLM 优先使用工具而非冗长叙述。

### Q3：todo 和后续的 task 系统有什么区别？

| 特性 | Todo (s03) | Task (s04+) |
|------|------------|-------------|
| 持久化 | 无（内存） | 无（子 Agent 返回即丢弃） |
| 上下文 | 共享主上下文 | 独立上下文 |
| 适用场景 | 任务进度追踪 | 复杂子任务委托 |
| 返回形式 | 渲染文本留在主上下文 | 摘要文本返回主上下文 |
| 复杂度 | 简单状态机 | 独立 Agent 循环 |

### Q4：为什么提醒注入到 results 而不是单独追加一条消息？

**答案**：注入到 `results` 列表的头部有几个好处：
1. 提醒与工具结果在同一条 `role: "user"` 消息中，LLM 可以自然地将两者关联起来
2. 使用 `insert(0, ...)` 确保提醒出现在工具结果之前，LLM 优先看到
3. 避免在 `messages` 中插入额外的消息对，保持消息结构的规整性（每轮只有一条 assistant + 一条 user）

---

## 9. 最佳实践

### 9.1 何时使用 todo

```
适合使用 todo：
- 多步骤任务（3+ 步骤）
- 需要追踪进度的任务
- 可能被打断的任务
- 需要向用户展示进度的场景

不需要 todo：
- 单步任务（如"读取文件"）
- 探索性任务（不确定步骤）
- 快速实验
```

### 9.2 编写好的任务描述

```python
# 模糊的任务
{"id": "1", "text": "处理代码", "status": "pending"}

# 明确的任务
{"id": "1", "text": "为 auth.py 的所有函数添加类型注解", "status": "pending"}
```

### 9.3 合理的任务粒度

```
层级 1（太粗）：["完成重构"]
层级 2（合适）：["添加类型注解", "添加文档字符串", "重命名函数"]
层级 3（太细）：["为函数 A 添加类型", "为函数 B 添加类型", ...]
```

**建议**：每个任务应该是可以在 1-3 轮内完成的独立单元。

### 9.4 正确的状态转换

```
pending --> in_progress --> completed
                                       （不存在 "done" 状态！）
```

记住：合法的状态值只有 `pending`、`in_progress`、`completed` 三种。

---

## 实战应用：让 Agent 拥有规划能力

前面的章节从原理层面解释了 TodoManager 的设计和实现。下面我们来看三个真实场景，以及如何在实际项目中做出正确的技术选择。

### 场景 1：多步骤部署流程

假设你需要 Agent 执行一个完整的 CI/CD 部署流程：

```
用户的指令：把最新的代码部署到生产环境

Agent 的规划：
[ ] #1: 构建项目（npm run build）
[ ] #2: 运行测试套件（npm test）
[ ] #3: 部署到生产服务器（deploy.sh）
[ ] #4: 健康检查（curl /health）
[ ] #5: 发送部署通知（notify.sh）

执行过程：
轮次 1：创建 todo 列表，#1 标记为 in_progress
轮次 2：执行构建，#1 完成，#2 标记为 in_progress
轮次 3：运行测试，#2 完成，#3 标记为 in_progress
轮次 4：部署到生产，#3 完成，#4 标记为 in_progress
轮次 5：健康检查通过，#4 完成，#5 标记为 in_progress
轮次 6：发送通知，#5 完成

关键价值：
- 部署中断时能准确知道停在哪一步
- 回滚时可以清晰地看到已完成的步骤
- 每一步的结果都在 todo 上下文中可追溯
```

### 场景 2：代码重构项目

```
用户的指令：把 auth.py 重构成模块化结构

Agent 的规划（10 个子任务，正好在 20 条上限之内）：
[ ] #1:  分析 auth.py 的函数列表和依赖关系
[ ] #2:  提取认证逻辑到 auth/core.py
[ ] #3:  提取 JWT 处理到 auth/jwt.py
[ ] #4:  提取权限检查到 auth/permissions.py
[ ] #5:  创建 auth/__init__.py 统一导出
[ ] #6:  更新所有 import 语句
[ ] #7:  运行现有测试验证行为不变
[ ] #8:  修复因重构引入的测试失败
[ ] #9:  为新模块添加单元测试
[ ] #10: 提交重构结果

关键价值：
- 重构是最容易"迷失"的任务类型——Agent 可能在修改 import 时
  忘记还有新模块的测试需要编写
- 单一 in_progress 约束迫使 Agent 逐步推进，不会跳步
- 完成度统计（如 4/10 completed）让用户随时了解进度
```

### 场景 3：数据迁移任务

```
用户的指令：把用户数据从旧数据库迁移到新数据库

Agent 的规划：
[ ] #1:  导出旧数据库的用户表（SELECT * INTO OUTFILE）
[ ] #2:  转换数据格式（CSV -> JSON，字段映射）
[ ] #3:  验证转换结果（行数一致性、字段完整性）
[ ] #4:  导入到新数据库（INSERT INTO）
[ ] #5:  回滚测试（验证可以安全回退）

关键价值：
- 数据迁移有严格的顺序依赖，规划确保不跳步
- 验证步骤（#3）和回滚测试（#5）容易遗漏，
  显式规划确保 Agent 不跳过关键安全步骤
- 如果迁移在 #4 失败，todo 状态告诉 Agent 已完成哪些步骤，
  辅助决策是继续还是回滚
```

### 决策树：什么时候用什么方案

面对一个多步骤任务时，按以下决策树选择方案：

```
任务开始
  |
  v
步骤数 <= 2？
  |-- 是 --> 直接执行（不需要规划，工具调用即可）
  |
  v
步骤数 3-20，且每步在同一上下文中？
  |-- 是 --> 使用 Todo（s03 方案）
  |         适合：文件重构、部署流程、数据迁移
  |
  v
步骤数 > 20，或需要读取大量文件？
  |-- 是 --> 考虑拆分为多个子任务
  |         每个子任务用子 Agent（s04 方案）
  |         主 Agent 用 Todo 追踪子任务进度
  |
  v
子任务之间有复杂依赖关系？
  |-- 是 --> 使用 Task 系统（s07 方案）
  |         支持：持久化、DAG 依赖、跨会话
  |
  v
当前方案：Todo（s03）
  |
  v
任务是否可能跨会话？
  |-- 是 --> 升级到 Task 系统（s07）
  |-- 否  --> 继续使用 Todo
```

**速查表**：

| 条件 | 推荐方案 |
|------|----------|
| 单步操作，如"读取这个文件" | 直接执行 |
| 3-20 步，同一上下文，不需要读大量文件 | Todo（s03） |
| 需要保持主上下文清洁（读大量文件） | 子 Agent（s04） |
| 超过 20 步，或有复杂依赖 | Task 系统（s07） |
| 需要跨会话持久化 | Task 系统（s07） |
| 规划 + 委托的组合 | Todo + 子 Agent（s03 + s04） |

---

## 10. 小结

### 10.1 核心要点

| 要点 | 说明 |
|------|------|
| **规划的必要性** | 多步骤任务需要显式计划，防止 Agent 随机游走 |
| **TodoManager** | 带严格校验的状态管理器（20 条上限、枚举校验、单焦点约束） |
| **三种合法状态** | `pending` / `in_progress` / `completed`（注意不是 `done`） |
| **单一 in_progress** | 强制聚焦单一任务，避免混乱 |
| **提醒机制** | 3 轮未调用 todo 则在 results 头部注入 `<reminder>` |
| **原子性更新** | 校验失败则整个更新被拒绝，状态不变 |

### 10.2 关键洞察

```
规划的核心价值：
1. 将复杂任务分解为可管理的步骤
2. 提供可视化的进度跟踪（含完成度统计）
3. 防止 Agent 在长对话中迷失

提醒机制的核心价值：
1. 纠正 Agent 的偏离行为
2. 保持计划的有效性
3. 提高任务完成率
4. 注入到 results 头部，确保优先被注意

防御性编程的核心价值：
1. 20 条上限防止 token 浪费
2. 枚举校验防止非法状态
3. 单焦点约束防止混乱
4. 原子性更新防止部分更新
```

---

## 11. 练习

### 练习 1：扩展 TodoManager

添加优先级字段（high/medium/low），并在渲染时用不同符号表示。注意同时修改工具的 JSON Schema。

### 练习 2：任务依赖

修改 TodoManager，支持 `depends_on` 字段，标记任务依赖关系。当依赖任务未完成时，阻止标记为 in_progress。

### 练习 3：子任务

允许任务包含子任务，形成层级结构。渲染时用缩进表示层级关系。

### 练习 4：提醒策略

实现不同的提醒策略：
- 固定轮次提醒（当前实现，阈值 = 3）
- 基于时间提醒（每 N 秒）
- 基于行为提醒（检测到偏离时）

### 练习 5：验证提醒效果

修改 `rounds_since_todo` 的阈值为 1、3、5，分别运行同一个多步骤任务，对比 Agent 的行为差异。

---

## 12. 下一步

现在 Agent 可以规划任务了。但所有操作都在一个上下文中，文件读取会快速消耗 Token。

在 [s04：子 Agent](./s04-subagent.md) 中，我们将学习：
- 如何将子任务委托给独立的 Agent
- 如何保持主上下文的清洁
- 父子 Agent 的通信模式和工具隔离

---

< 上一章：[s02-工具系统](./s02-tool-use.md) | [目录](./index.md) | 下一章：[s04-子 Agent](./s04-subagent.md) >
