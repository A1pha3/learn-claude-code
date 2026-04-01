# s03：任务规划 -- 让 Agent 不再迷失

> **核心口号**：*"没有计划的 Agent 会迷失"*

> **学习目标**：理解规划的重要性，实现任务清单系统，掌握状态机设计

---

## 学习目标

完成本章后，你将能够：

1. **理解为什么 Agent 需要规划** -- 避免"迷失方向"的问题
2. **实现 TodoManager** -- 一个带严格校验的任务状态管理器
3. **掌握提醒机制** -- 当 Agent 偏离时自动注入提醒
4. **理解状态机设计** -- pending / in_progress / completed 三态与单一焦点约束
5. **掌握上限防护** -- 20 条上限、状态枚举校验等防御性编程技巧

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

这迫使 Agent **聚焦单一任务**，避免同时做多件事导致混乱。源码第 68-72 行实现了这一约束：

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

| 方面 | s02 | s03 |
|------|-----|-----|
| 工具数量 | 4 | 5 (+todo) |
| 规划能力 | 无 | TodoManager（全局单例） |
| 状态管理 | 无 | pending / in_progress / completed |
| 提醒机制 | 无 | 3 轮后自动注入 reminder 到 results |
| 循环结构 | `while stop_reason == "tool_use"` | 相同 + `rounds_since_todo` 计数器 |
| 校验能力 | 基础错误处理 | TodoManager 内建多层校验 |

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
