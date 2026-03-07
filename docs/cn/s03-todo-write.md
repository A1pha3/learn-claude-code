# s03：任务规划 —— 让 Agent 不再迷失

> **核心口号**：*"没有计划的 Agent 会迷失"*

> **学习目标**：理解规划的重要性，实现任务清单系统，掌握状态机设计

---

## 学习目标

完成本章后，你将能够：

1. **理解为什么 Agent 需要规划** —— 避免"迷失方向"的问题
2. **实现 TodoManager** —— 一个简单的任务状态管理器
3. **掌握提醒机制** —— 当 Agent 偏离时自动提醒
4. **理解状态机设计** —— 单一 in_progress 约束的意义

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
┌─────────────────────────────────────┐
│         LLM 的上下文窗口            │
│                                     │
│  [系统提示]                         │
│  [用户提问]                         │
│  [工具调用 1: read_file]            │
│  [工具结果 1: ...5000 tokens...]   │ ← 大量输出
│  [工具调用 2: read_file]            │
│  [工具结果 2: ...5000 tokens...]   ← 又大量输出
│  [工具调用 3: run_bash]             │
│  [工具结果 3: ...3000 tokens...]   │
│                                     │
│  ═══ 系统提示被淹没 ═══             │
│  ═══ 原始计划被遗忘 ═══             │
└─────────────────────────────────────┘
```

随着对话进行，工具结果堆积，早期的重要内容（如系统提示和用户指令）被新内容"挤"出注意力。

### 1.3 解决方案：显式规划

```
┌─────────────────────────────────────┐
│         TodoManager 状态            │
│                                     │
│  [ ] 1. 添加类型注解                │
│  [>] 2. 添加文档字符串 ← 正在做     │
│  [ ] 3. 重命名函数                  │
│  [ ] 4. 提取验证逻辑                │
│  [ ] 5. 添加单元测试                │
│  [x] 6. 更新导入 ← 已完成           │
│  [ ] 7. 运行测试                    │
│  [ ] 8. 修复测试失败                │
│  [ ] 9. 更新文档                    │
│  [ ] 10. 提交更改                   │
│                                     │
│  每轮都可见，不会被淹没              │
└─────────────────────────────────────┘
```

**关键点**：Todo 状态通过工具结果持续注入到上下文中，确保 Agent 始终能看到计划。

---

## 2. TodoManager 设计

### 2.1 数据结构

```python
from typing import List, Literal

TodoStatus = Literal["pending", "in_progress", "done"]

class TodoItem:
    id: str           # 唯一标识
    text: str         # 任务描述
    status: TodoStatus  # 状态

# 示例
{"id": "1", "text": "添加类型注解", "status": "pending"}
{"id": "2", "text": "添加文档字符串", "status": "in_progress"}
{"id": "3", "text": "重命名函数", "status": "done"}
```

### 2.2 状态定义

| 状态 | 含义 | 符号 | 约束 |
|------|------|------|------|
| `pending` | 待办 | `[ ]` | 可以有多个 |
| `in_progress` | 进行中 | `[>]` | **只能有一个** |
| `done` | 完成 | `[x]` | 可以有多个 |

**为什么只能有一个 in_progress？**

这迫使 Agent **聚焦单一任务**，避免同时做多件事导致混乱。

```
❌ 多个 in_progress（混乱）：
[>] 任务 A
[>] 任务 B  ← 同时做两件事？
[>] 任务 C
[ ] 任务 D

✅ 单一 in_progress（清晰）：
[x] 任务 A
[>] 任务 B  ← 专注当前任务
[ ] 任务 C
[ ] 任务 D
```

### 2.3 TodoManager 实现

```python
from typing import List, Dict, Literal

TodoStatus = Literal["pending", "in_progress", "done"]

class TodoManager:
    def __init__(self):
        self.items: List[Dict] = []

    def update(self, items: List[Dict]) -> str:
        """更新任务列表"""
        # 验证
        in_progress_count = 0
        validated = []

        for item in items:
            if "id" not in item or "text" not in item:
                continue  # 跳过无效项

            status = item.get("status", "pending")
            if status not in ["pending", "in_progress", "done"]:
                status = "pending"

            if status == "in_progress":
                in_progress_count += 1

            validated.append({
                "id": item["id"],
                "text": item["text"],
                "status": status
            })

        # 约束：只能有一个 in_progress
        if in_progress_count > 1:
            raise ValueError("只能有一个任务处于 in_progress 状态")

        self.items = validated
        return self.render()

    def render(self) -> str:
        """渲染任务列表为文本"""
        if not self.items:
            return "无任务"

        lines = []
        for item in self.items:
            status = item["status"]
            text = item["text"]

            if status == "pending":
                symbol = "[ ]"
            elif status == "in_progress":
                symbol = "[>]"
            else:  # done
                symbol = "[x]"

            lines.append(f"{symbol} {item['id']}. {text}")

        return "\n".join(lines)

    def get_summary(self) -> str:
        """获取摘要"""
        total = len(self.items)
        done = sum(1 for i in self.items if i["status"] == "done")
        in_progress = sum(1 for i in self.items if i["status"] == "in_progress")

        return f"任务进度：{done}/{total} 完成，{in_progress} 进行中"
```

---

## 3. 集成到 Agent

### 3.1 添加 todo 工具

```python
TOOLS = [
    # ... 其他工具 ...
    {
        "name": "todo",
        "description": "更新任务列表。必须确保只有一个任务处于 in_progress 状态。",
        "input_schema": {
            "type": "object",
            "properties": {
                "items": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "properties": {
                            "id": {"type": "string"},
                            "text": {"type": "string"},
                            "status": {
                                "type": "string",
                                "enum": ["pending", "in_progress", "done"]
                            }
                        },
                        "required": ["id", "text"]
                    }
                }
            },
            "required": ["items"]
        }
    }
]
```

### 3.2 注册处理器

```python
TODO = TodoManager()

TOOL_HANDLERS = {
    # ... 其他工具 ...
    "todo": lambda **kw: TODO.update(kw["items"]),
}
```

### 3.3 提醒机制

当 Agent 长时间不更新 todo 时，自动注入提醒：

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]
    rounds_since_todo = 0

    while True:
        # 微压缩：每轮都执行
        micro_compact(messages)

        # 提醒机制：3 轮没有调用 todo 就提醒
        if rounds_since_todo >= 3 and messages:
            last_msg = messages[-1]
            if last_msg["role"] == "user":
                if isinstance(last_msg.get("content"), list):
                    # 在工具结果前插入提醒
                    last_msg["content"].insert(0, {
                        "type": "text",
                        "text": "<reminder>请更新你的任务列表（todo 工具）。</reminder>"
                    })
            rounds_since_todo = 0  # 重置计数器

        response = client.messages.create(
            model=MODEL,
            messages=messages,
            tools=TOOLS,
            max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            break

        # 检查是否调用了 todo
        todo_called = False
        results = []
        for block in response.content:
            if block.type == "tool_use":
                if block.name == "todo":
                    todo_called = True
                # 执行工具...
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input)
                results.append({...})

        messages.append({"role": "user", "content": results})

        # 更新计数器
        if todo_called:
            rounds_since_todo = 0
        else:
            rounds_since_todo += 1

    return response.content
```

---

## 4. 完整示例

### 4.1 用户请求

```
重构 auth.py：添加类型注解、文档字符串、重命名 login_to_system 为 login
```

### 4.2 Agent 执行流程

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
    [ ] 1. 添加类型注解
    [ ] 2. 添加文档字符串
    [ ] 3. 重命名函数

轮次 2：
  LLM：现在开始第一个任务
        tool_use: todo(items=[...1: "in_progress"...])
        tool_use: read_file("auth.py")
  工具结果：
    [>] 1. 添加类型注解
    [ ] 2. 添加文档字符串
    [ ] 3. 重命名函数
    [文件内容...]

轮次 3：
  LLM：添加类型注解
        tool_use: write_file(...)
  工具结果：已写入文件

轮次 4：
  LLM：完成第一个任务，开始第二个
        tool_use: todo(items=[...1: "done", 2: "in_progress"...])
  工具结果：
    [x] 1. 添加类型注解
    [>] 2. 添加文档字符串
    [ ] 3. 重命名函数

轮次 5：
  LLM：添加文档字符串
        tool_use: edit_file(...)

轮次 6：
  LLM：完成所有任务
        tool_use: todo(items=[...all: "done"...])
  工具结果：
    [x] 1. 添加类型注解
    [x] 2. 添加文档字符串
    [x] 3. 重命名函数

轮次 7：
  LLM：重构已完成！
  返回
```

---

## 5. 提醒机制详解

### 5.1 为什么需要提醒

即使有 todo 工具，Agent 也可能"忘记"更新它：

```
轮次 1：创建 todo（in_progress = 任务 A）
轮次 2：完成任务 A
轮次 3：开始任务 B ← 忘记更新 todo
轮次 4：继续任务 B
轮次 5：继续任务 B ← 提醒！更新你的 todo
```

### 5.2 提醒的实现方式

```python
# 方式 1：在消息中插入提醒
if rounds_since_todo >= 3:
    messages[-1]["content"].insert(0, {
        "type": "text",
        "text": "<reminder>请更新你的任务列表。</reminder>"
    })

# 方式 2：作为系统消息
if rounds_since_todo >= 3:
    messages.append({
        "role": "user",
        "content": "<reminder>请更新你的任务列表。</reminder>"
    })
    messages.append({
        "role": "assistant",
        "content": "我会更新任务列表。"
    })
```

**方式 1 更好**：提醒作为工具结果的一部分，LLM 更容易理解这是关于工具使用的反馈。

### 5.3 提醒频率的权衡

| 轮次阈值 | 优点 | 缺点 |
|----------|------|------|
| 1-2 | 从不偏离 | 提醒过于频繁 |
| 3-5 | 平衡 | 可能有些延迟 |
| 6+ | 不打扰 | 可能严重偏离 |

**推荐**：3-5 轮是一个好的平衡点。

---

## 6. 与 s02 的对比

| 方面 | s02 | s03 |
|------|-----|-----|
| 工具数量 | 4 | 5 (+todo) |
| 规划能力 | 无 | TodoManager |
| 状态管理 | 无 | pending/in_progress/done |
| 提醒机制 | 无 | 3 轮后自动提醒 |
| 循环结构 | `while stop_reason` | 相同（增加了计数器） |

---

## 7. 常见问题

### Q1：为什么不用数据库？

```python
# 为什么用内存列表而不是数据库？
self.items: List[Dict] = []  # 简单列表
```

**答案**：
1. TodoManager 是**短期记忆**，不需要跨会话持久化
2. 列表更简单，序列化到消息中很方便
3. s07 会引入持久化的任务系统

### Q2：可以跳过规划直接执行吗？

**答案**：可以，但不推荐。对于简单任务（如"读取文件"），直接执行更高效。但对于复杂任务（3+ 步骤），规划能显著提高成功率。

### Q3：todo 和后续的 task 系统有什么区别？

| 特性 | Todo (s03) | Task (s07) |
|------|------------|------------|
| 持久化 | 无（内存） | 有（磁盘） |
| 依赖关系 | 无 | 有（DAG） |
| 适用场景 | 单次会话 | 跨会话项目 |
| 复杂度 | 简单 | 复杂 |

---

## 8. 最佳实践

### 8.1 何时使用 todo

```
✅ 适合使用 todo：
- 多步骤任务（3+ 步骤）
- 需要追踪进度的任务
- 可能被打断的任务

❌ 不需要 todo：
- 单步任务（如"读取文件"）
- 探索性任务（不确定步骤）
- 快速实验
```

### 8.2 编写好的任务描述

```python
# ❌ 模糊的任务
{"id": "1", "text": "处理代码", "status": "pending"}

# ✅ 明确的任务
{"id": "1", "text": "为 auth.py 的所有函数添加类型注解", "status": "pending"}
```

### 8.3 合理的任务粒度

```
层级 1（太粗）：["完成重构"]
层级 2（合适）：["添加类型注解", "添加文档字符串", "重命名函数"]
层级 3（太细）：["为函数 A 添加类型", "为函数 B 添加类型", ...]
```

**建议**：每个任务应该是可以在 1-3 轮内完成的独立单元。

---

## 9. 小结

### 9.1 核心要点

| 要点 | 说明 |
|------|------|
| **规划的必要性** | 多步骤任务需要显式计划 |
| **TodoManager** | 简单的状态管理器 |
| **单一 in_progress** | 强制聚焦单一任务 |
| **提醒机制** | 防止 Agent 偏离计划 |

### 9.2 代码模板

```python
# 1. 定义 TodoManager
TODO = TodoManager()

# 2. 添加 todo 工具
TOOLS.append({"name": "todo", ...})

# 3. 注册处理器
TOOL_HANDLERS["todo"] = lambda **kw: TODO.update(kw["items"])

# 4. 添加提醒机制
if rounds_since_todo >= 3:
    inject_reminder(messages)
```

### 9.3 关键洞察

```
规划的核心价值：
1. 将复杂任务分解为可管理的步骤
2. 提供可视化的进度跟踪
3. 防止 Agent 在长对话中迷失

提醒机制的核心价值：
1. 纠正 Agent 的偏离行为
2. 保持计划的有效性
3. 提高任务完成率
```

---

## 10. 练习

### 练习 1：扩展 TodoManager

添加优先级字段（高/中/低），并在渲染时用不同符号表示。

### 练习 2：任务依赖

修改 TodoManager，支持 `depends_on` 字段，标记任务依赖关系。

### 练习 3：子任务

允许任务包含子任务，形成层级结构。

### 练习 4：提醒策略

实现不同的提醒策略：
- 固定轮次提醒（当前实现）
- 基于时间提醒（每 N 秒）
- 基于行为提醒（检测到偏离时）

---

## 11. 下一步

现在 Agent 可以规划任务了。但所有操作都在一个上下文中，文件读取会快速消耗 Token。

在 [s04：子 Agent](./s04-subagent.md) 中，我们将学习：
- 如何将子任务委托给独立的 Agent
- 如何保持主上下文的清洁
- 父子 Agent 的通信模式

---

< 上一章：[s02-工具系统](./s02-tool-use.md) | [目录](./index.md) | 下一章：[s04-子 Agent](./s04-subagent.md) >
