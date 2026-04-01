# s04：子 Agent -- 上下文隔离的艺术

> **核心口号**：*"进程隔离天然带来上下文隔离"*

> **学习目标**：理解上下文隔离的必要性，掌握子 Agent 模式，学会保持主对话的清洁

---

## 学习目标

完成本章后，你将能够：

1. **理解上下文污染问题** -- 为什么长对话会"变笨"
2. **掌握子 Agent 模式** -- 委托子任务给独立 Agent
3. **理解消息隔离** -- 父子 Agent 的独立上下文
4. **学会结果摘要** -- 将复杂过程浓缩为简洁结论
5. **理解工具隔离** -- 子 Agent 的工具集为何不包含 task，防止递归

### 学习分层指南

| 层级 | 掌握标准 | 对应内容 | 建议用时 |
|------|----------|----------|----------|
| L1 基础 | 能说明上下文污染问题，能运行示例并观察父子 Agent 的上下文隔离效果 | 1.问题的本质、2.子 Agent 模式、4.完整执行流程示例 | 30 分钟 |
| L2 进阶 | 能独立实现 run_subagent 函数，能解释工具隔离和结果摘要的精确机制 | 3.子 Agent 实现、5.上下文隔离的价值、6.架构分析 | 45 分钟 |
| L3 专家 | 能设计多层子 Agent 调度策略，能结合 s03 TodoManager 实现规划+委托的完整工作流 | 8.最佳实践、10.练习 | 60 分钟 |

---

## 0. 上手演练（建议先做）

先运行子 Agent 示例，再理解上下文隔离机制：

```bash
uv run python agents/s04_subagent.py
```

建议观察：

1. 当主 Agent 调用 task 工具时，控制台会打印 `> task (desc): prompt...`，表示子 Agent 被派发。
2. 子 Agent 执行过程中产生的工具调用和结果**不会**出现在主上下文中。
3. 最终只有摘要文本返回给主 Agent，观察主 Agent 后续对话是否基于这个摘要。
4. 尝试让主 Agent 调用子 Agent 去读取多个大文件，对比不使用子 Agent 时主上下文的 token 消耗。

---

## 1. 问题的本质

### 1.1 上下文污染

想象一个典型的编程任务：

```
用户：这个项目用什么测试框架？

Agent 的思考过程：
1. 读取 setup.py（1000 tokens）
2. 读取 pyproject.toml（500 tokens）
3. 读取 pytest.ini（200 tokens）
4. 搜索 test 目录（300 tokens）
5. 读取 tests/conftest.py（800 tokens）
6. 检查 requirements.txt（200 tokens）

结论：项目使用 pytest

消耗：约 3000 tokens
主上下文：增加了 6 条工具调用 + 6 条结果
```

**问题**：为了回答一个简单问题，主上下文被大量中间细节污染。

### 1.2 污染的后果

```
主 Agent 的上下文窗口：

+-------------------------------------------+
| [系统提示]                                 | <-- 重要但被淹没
| [用户问题]                                 |
| [工具调用 1] read_file("setup.py")         |
| [工具结果 1] ...1000 tokens...             | <-- 大量细节
| [工具调用 2] read_file("pyproject.toml")   |
| [工具结果 2] ...500 tokens...              |
| [工具调用 3] read_file("pytest.ini")       |
| [工具结果 3] ...200 tokens...              |
| [工具调用 4] ...                           |
| [工具结果 4] ...                           |
| [工具调用 5] ...                           |
| [工具结果 5] ...                           |
| [工具调用 6] ...                           |
| [工具结果 6] ...200 tokens...              |
| [LLM 结论] 项目使用 pytest                 | <-- 真正需要的
+-------------------------------------------+

系统提示 + 用户问题被 3000+ tokens 的细节挤出注意力
```

**后果**：
1. 系统提示的指令被遗忘
2. Token 预算被快速消耗
3. 后续决策基于不完整的信息

### 1.3 理想的解决方案

```
主 Agent：
  用户：这个项目用什么测试框架？
  主 Agent：让子 Agent 去调查
    |-> 子 Agent：
        1. 读取 setup.py
        2. 读取 pyproject.toml
        3. 搜索测试文件
        4. 分析并总结
        |-> 返回："项目使用 pytest，配置在 pytest.ini 中"

  主 Agent：收到简洁的答案（约 50 tokens）
  上下文保持清洁
```

**核心思想**：子 Agent 承担"脏活累活"，主 Agent 只看结论。

---

## 2. 子 Agent 模式

### 2.1 架构对比

```
没有子 Agent（单 Agent）：
+-------------------------------------+
|         主 Agent                     |
|                                     |
|  1. read_file("a.py")  -> 大量输出  |
|  2. read_file("b.py")  -> 大量输出  |
|  3. read_file("c.py")  -> 大量输出  |
|  4. read_file("d.py")  -> 大量输出  |
|  5. read_file("e.py")  -> 大量输出  |
|  6. 总结                             |
|                                     |
|  所有细节都在主上下文中               |
+-------------------------------------+

有子 Agent（父子模式）：
+------------------+
|    主 Agent      |
|                  |
|  1. task("分析...")  | --------+
|  2. 收到摘要         |        |
|                  |        v
|  只看到结论      |   +-----------+
|                  |   | 子 Agent  |
+------------------+   |           |
                       | 1. 读取 5 个文件
                       | 2. 分析
                       | 3. 返回摘要
                       +-----------+
```

### 2.2 父子 Agent 的工具分配

源码中工具分配的核心机制（第 94-140 行）：

```python
# 共享的工具处理器字典（父和子都用这个来路由工具调用）
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}

# 子 Agent 的工具集：4 个基础工具，不含 task（防止递归派发）
CHILD_TOOLS = [
    {"name": "bash", ...},
    {"name": "read_file", ...},
    {"name": "write_file", ...},
    {"name": "edit_file", ...},
]

# 父 Agent 的工具集：子 Agent 工具 + task 工具
PARENT_TOOLS = CHILD_TOOLS + [
    {"name": "task", "description": "Spawn a subagent with fresh context. ...",
     "input_schema": {
         "properties": {
             "prompt": {"type": "string"},
             "description": {"type": "string", "description": "Short description of the task"}
         },
         "required": ["prompt"]
     }},
]
```

**关键区别**：

| 工具 | 子 Agent | 父 Agent |
|------|----------|----------|
| `bash` | 有 | 有 |
| `read_file` | 有 | 有 |
| `write_file` | 有 | 有 |
| `edit_file` | 有 | 有 |
| `task` | **无** | **有** |
| `todo` | **无** | **无** |

> **注意**：s04 的源码中，子 Agent **不包含** `todo` 工具。`CHILD_TOOLS` 只有 4 个基础工具（bash、read_file、write_file、edit_file）。`todo` 工具是 s03 的特性，s04 并未集成。

**为什么子 Agent 不能有 task 工具？**

防止无限递归：子 Agent 创建子子 Agent，子子 Agent 创建子子子 Agent... 源码通过 `PARENT_TOOLS = CHILD_TOOLS + [task]` 的列表拼接方式，在结构上保证了子 Agent 无法获取 task 工具。

### 2.3 task 工具的参数

源码第 138-140 行定义了 task 工具的完整 Schema：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `prompt` | string | 是 | 传给子 Agent 的详细指令，成为子 Agent 的初始 user 消息 |
| `description` | string | 否 | 任务的简短描述，用于控制台打印 |

`description` 仅用于父 Agent 的控制台日志（源码第 156 行 `desc = block.input.get("description", "subtask")`），不影响子 Agent 的行为。

---

## 3. 子 Agent 实现

### 3.1 核心函数 run_subagent()

源码第 115-133 行是子 Agent 的完整实现：

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]  # 干净的上下文
    for _ in range(30):  # 硬编码的安全上限
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM, messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        sub_messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
    # 只返回最终文本摘要 -- 子 Agent 的整个上下文被丢弃
    return "".join(b.text for b in response.content if hasattr(b, "text")) or "(no summary)"
```

### 3.2 关键设计点逐项分析

**1. 干净的开始**

```python
sub_messages = [{"role": "user", "content": prompt}]
# 不是：sub_messages = parent_messages.copy()
```

子 Agent 不继承父 Agent 的历史。它收到的唯一消息就是 `prompt`，这个 prompt 来自父 Agent 调用 task 工具时的 `block.input["prompt"]`。

**2. 独立的系统提示**

源码第 42 行：

```python
SUBAGENT_SYSTEM = f"You are a coding subagent at {WORKDIR}. Complete the given task, then summarize your findings."
```

子 Agent 的系统提示与父 Agent 不同（源码第 41 行）：

```python
SYSTEM = f"You are a coding agent at {WORKDIR}. Use the task tool to delegate exploration or subtasks."
```

对比：

| 方面 | 父 Agent (SYSTEM) | 子 Agent (SUBAGENT_SYSTEM) |
|------|-------------------|---------------------------|
| 角色 | coding agent | coding subagent |
| 核心指令 | 使用 task 工具委托任务 | 完成给定任务并总结 |
| 工具集 | 含 task | 不含 task |

**3. 轮次限制**

```python
for _ in range(30):  # 硬编码的循环上限
```

源码中没有定义 `MAX_SUB_ROUNDS` 常量，而是在 `range(30)` 中硬编码了数值 30。这意味着子 Agent 最多执行 30 轮工具调用循环。这是一个安全机制，防止子 Agent 陷入无限循环。

**4. 最终文本摘要的提取**

源码第 133 行：

```python
return "".join(b.text for b in response.content if hasattr(b, "text")) or "(no summary)"
```

这行代码的关键逻辑：

- `response.content` 是一个 Block 列表，可能包含 `text` 类型的 Block 和 `tool_use` 类型的 Block
- `hasattr(b, "text")` 过滤出所有文本 Block（排除 tool_use Block）
- `"".join(...)` 将所有文本 Block 拼接成一个字符串
- `or "(no summary)"` 处理空字符串的情况，确保始终返回非空值

**重要细节**：这里取的是**最后一次** LLM 响应中的文本，而非所有轮次的文本。子 Agent 在中间轮次产生的大量工具调用和结果全部被丢弃。

**5. 工具结果截断**

```python
results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(output)[:50000]})
```

子 Agent 的工具结果被截断到 50000 字符，与父 Agent 的工具处理一致。

### 3.3 父 Agent 的 task 工具调度

源码第 143-164 行的 `agent_loop` 中，task 工具的调度逻辑在第 155-158 行：

```python
if block.name == "task":
    desc = block.input.get("description", "subtask")
    print(f"> task ({desc}): {block.input['prompt'][:80]}")
    output = run_subagent(block.input["prompt"])
```

调度流程：

```
父 Agent 产生 tool_use block
    |
    v
block.name == "task" ?
    |
    +-- 是 --> 从 block.input 取 prompt 和 description
    |          打印日志到控制台
    |          调用 run_subagent(prompt)
    |          将返回的摘要作为 tool_result
    |
    +-- 否 --> 查找 TOOL_HANDLERS 中的处理器
               执行并获取输出
    |
    v
将输出追加到 results 列表
results 作为 user 消息追加到 messages
```

---

## 4. 完整执行流程示例

### 4.1 用户请求

```
用户：帮我分析这个项目的架构，找出所有模块及其职责
```

### 4.2 主 Agent 的处理

```
轮次 1（主 Agent）：
  用户：分析项目架构...
  主 Agent：这个任务需要读取多个文件，让我用子 Agent
        tool_use: task(
          prompt="分析 /workspace 目录下的 Python 模块，
                  找出所有 .py 文件，分析它们的职责和依赖关系",
          description="分析项目架构"
        )

  控制台输出：> task (分析项目架构): 分析 /workspace 目录下的 Python 模块，找出所有 .py 文件，分析它们...

  [子 Agent 开始运行...]
```

### 4.3 子 Agent 的处理

```
子 Agent（独立上下文，sub_messages = [{"role": "user", "content": prompt}]）：

  子 Agent 轮次 1：
    tool_use: bash(command="find . -name '*.py' -type f")
    结果：[20 个文件的列表]

  子 Agent 轮次 2-10：
    tool_use: read_file(path="auth.py")
    tool_use: read_file(path="database.py")
    tool_use: read_file(path="api.py")
    ... 读取并分析所有文件

  子 Agent 轮次 11：
    stop_reason = "end_turn"
    返回文本：
      "项目包含以下模块：
       1. auth.py - 用户认证和授权
       2. database.py - 数据库连接和操作
       3. api.py - REST API 端点
       4. utils.py - 通用工具函数
       ...
       模块依赖：api -> auth -> database"
```

### 4.4 主 Agent 收到摘要

```
轮次 1 继续（主 Agent）：
  tool_result：[子 Agent 的摘要文本]
  内容："项目包含以下模块：1. auth.py - 用户认证..."

轮次 2（主 Agent）：
  主 Agent：根据子 Agent 的分析，项目架构如下：
    [简洁地展示结果]

  上下文保持清洁：主 Agent 只消耗了约 600 tokens
  （task 调用约 100 tokens + 摘要约 500 tokens）
```

---

## 5. 上下文隔离的价值

### 5.1 Token 消耗对比

```
没有子 Agent：
  读取 20 个文件 x 平均 500 tokens = 10,000 tokens
  加上工具调用和中间处理 = 约 15,000 tokens

有子 Agent：
  task 调用 = 约 100 tokens
  子 Agent 摘要 = 约 500 tokens
  主 Agent 只消耗 = 约 600 tokens

节省：约 14,400 tokens（96%）
```

### 5.2 系统提示的保持

```
没有子 Agent：
  系统提示：[1000 tokens]
  工具结果：[15000 tokens]
  -------------------------------
  系统提示占比：1000/16000 = 6%（可能被忽略）

有子 Agent：
  系统提示：[1000 tokens]
  子 Agent 摘要：[500 tokens]
  -------------------------------
  系统提示占比：1000/1500 = 67%（保持有效）
```

### 5.3 进程隔离与上下文隔离

源码开头的注释（第 22 行）总结了核心设计洞察：

> **Key insight**: "Process isolation gives context isolation for free."

这里的"进程隔离"是一种比喻：子 Agent 拥有独立的 `sub_messages` 列表（相当于独立的"进程空间"），与父 Agent 的 `messages` 列表完全隔离。子 Agent 执行完毕后，`sub_messages` 被丢弃，只把摘要文本带回。

```
父 Agent 进程空间：             子 Agent 进程空间：
+---------------------+        +---------------------+
| messages = [...]     |        | sub_messages = []   | <-- 新建
|                     | 派发    |                     |
| tool: task          | -----> | while tool_use:     |
|   prompt="..."      |        |   调用工具           |
|                     |        |   追加结果           |
|                     | 摘要    |                     |
| result = "摘要文本"  | <----- | return 最后的文本    |
+---------------------+        +---------------------+
         |
父上下文保持清洁。              子上下文被丢弃。
```

---

## 6. 架构分析

### 6.1 工具处理器共享

源码中父 Agent 和子 Agent 共享同一套工具处理器（`TOOL_HANDLERS`），但通过不同的工具 Schema 列表（`PARENT_TOOLS` vs `CHILD_TOOLS`）来控制可用工具：

```
TOOL_HANDLERS（共享）
    |
    +-- bash       -> run_bash()
    +-- read_file  -> run_read()
    +-- write_file -> run_write()
    +-- edit_file  -> run_edit()
    |
    +-- task       -> run_subagent()  [仅父 Agent 通过 agent_loop 直接调用]

CHILD_TOOLS = [bash, read_file, write_file, edit_file]           (4 个)
PARENT_TOOLS = CHILD_TOOLS + [task]                               (5 个)
```

注意：`TOOL_HANDLERS` 字典中**没有** `"task"` 键。在父 Agent 的 `agent_loop` 中，task 工具的调用是通过 `if block.name == "task":` 条件分支处理的（源码第 155 行），而非通过 `TOOL_HANDLERS` 路由。

### 6.2 安全防护

| 防护层 | 位置 | 机制 |
|--------|------|------|
| 路径穿越 | `safe_path()` | `resolve()` + `is_relative_to(WORKDIR)` 检查 |
| 危险命令 | `run_bash()` | 黑名单拦截 `rm -rf /`, `sudo`, `shutdown` 等 |
| 超时保护 | `run_bash()` | `subprocess.TimeoutExpired`，120 秒超时 |
| 输出截断 | 多处 | 50000 字符限制 |
| 递归防护 | 工具分配 | `CHILD_TOOLS` 不含 task，结构上阻止嵌套派发 |
| 循环上限 | `run_subagent()` | `range(30)` 硬编码，防止无限循环 |
| 无效工具 | `run_subagent()` | `TOOL_HANDLERS.get()` 返回 None 时返回 "Unknown tool" |

### 6.3 与 s03 的对比

| 方面 | s03 | s04 |
|------|-----|-----|
| 规划方式 | TodoManager（内存状态机） | 子 Agent（独立上下文） |
| 上下文管理 | 单一共享上下文 | 父子隔离 |
| 工具数量 | 5（含 todo） | 4（子）+ 1（父: task）= 5 |
| 结果返回 | 渲染文本留在主上下文 | 摘要文本返回主上下文 |
| Token 效率 | 中等（所有细节在主上下文） | 高（细节在子上下文中被丢弃） |
| 递归风险 | 无 | 需防护（通过工具隔离解决） |
| 提醒机制 | 有（rounds_since_todo >= 3） | 无 |

---

## 7. 常见问题

### Q1：子 Agent 和工具有什么区别？

| 特性 | 普通工具 | 子 Agent (task) |
|------|----------|-----------------|
| 能力 | 单一操作 | 多步骤推理 + 工具调用 |
| 上下文 | 共享主上下文 | 独立上下文 |
| 返回 | 原始工具输出 | 最终文本摘要 |
| 循环 | 无 | 最多 30 轮 |
| 路由 | 通过 TOOL_HANDLERS | 通过 agent_loop 中的条件分支 |
| 示例 | `read_file("a.py")` | `task(prompt="分析所有测试文件")` |

**本质区别**：子 Agent 是"有 LLM 的工具"，可以执行需要多步推理的任务。

### Q2：子 Agent 可以访问文件吗？

**答案**：可以。子 Agent 和父 Agent 共享同一套文件操作工具（`read_file`、`write_file`、`edit_file`）和 `bash` 工具，因为它们共享同一个 `WORKDIR`。区别在于子 Agent 在独立的上下文中运行。

### Q3：如何控制子 Agent 的行为？

**答案**：通过以下两个参数：

1. **系统提示**（`SUBAGENT_SYSTEM`，源码第 42 行）：定义子 Agent 的角色和行为准则
2. **prompt 参数**：具体任务指令，成为子 Agent 的第一条（也是唯一一条）user 消息

```python
# 源码中的子 Agent 系统提示
SUBAGENT_SYSTEM = f"You are a coding subagent at {WORKDIR}. Complete the given task, then summarize your findings."
```

可通过修改 `SUBAGENT_SYSTEM` 来适配不同场景：

```python
# 严格模式：只执行任务，不提出问题
SUBAGENT_SYSTEM = "你只执行任务，不提出问题。完成后返回摘要。"

# 调试模式：详细记录思考过程
SUBAGENT_SYSTEM = "详细记录你的思考过程，返回完整的推理链。"
```

### Q4：子 Agent 失败了怎么办？

源码中 `run_subagent` 没有额外的异常处理。如果在子 Agent 内部发生异常（例如工具调用失败），异常会通过 `TOOL_HANDLERS` 的返回值（如 `"Error: ..."`）传递，子 Agent 仍然会继续运行并尝试处理。

如果需要更健壮的错误处理，可以在父 Agent 的 `agent_loop` 中包裹 try/except：

```python
# 源码中未实现，但可以添加
try:
    output = run_subagent(block.input["prompt"])
except Exception as e:
    output = f"子任务执行失败: {str(e)}"
# 将错误作为摘要返回，主 Agent 可以决定如何处理
```

### Q5：为什么 task 工具不由 TOOL_HANDLERS 路由？

源码中 `TOOL_HANDLERS` 字典没有 `"task"` 键。task 工具在父 Agent 的 `agent_loop` 中通过 `if block.name == "task":` 条件分支处理（源码第 155 行）。

**原因**：task 工具的处理逻辑比普通工具复杂得多（需要创建独立的 Agent 循环），不适合用简单的 lambda 表达式来路由。此外，这种设计使得 `TOOL_HANDLERS` 可以被父 Agent 和子 Agent 安全共享，无需担心子 Agent 意外调用 task。

---

## 8. 最佳实践

### 8.1 何时使用子 Agent

```
适合使用子 Agent：
- 需要读取多个文件（3+）
- 复杂的分析任务
- 需要多步推理的子任务
- 想要保持主上下文清洁
- 探索性任务（不确定需要多少轮）

不需要子 Agent：
- 简单的单步操作
- 快速文件读取（1-2 个文件）
- 直接的工具调用就足够
```

### 8.2 编写好的子任务提示

```python
# 模糊的提示
task(prompt="分析代码")

# 明确的提示（推荐）
task(prompt="""
分析 /workspace 目录的 Python 代码结构：
1. 找出所有 .py 文件
2. 分析每个文件的主要类和函数
3. 识别模块之间的依赖关系
4. 返回简洁的架构摘要
""")
```

好的 prompt 应该：
- 明确任务的目标和范围
- 列出期望的步骤
- 指定输出的格式

### 8.3 子 Agent 的轮次控制

源码中子 Agent 的最大轮次硬编码为 30。对于不同的使用场景，可以考虑动态调整：

```python
# 简单任务：少轮次
max_rounds = 10

# 复杂任务：多轮次
max_rounds = 30

# 动态调整（根据提示长度）
max_rounds = min(30, 10 + len(prompt) // 100)
```

---

## 实战应用：识别项目中的子 Agent 机会

前面章节从原理层面解释了子 Agent 的设计和实现。下面通过三个真实场景（含 token 消耗对比数据）帮助你判断什么时候该用子 Agent。

### 场景 1：代码库架构分析

```
任务：分析一个包含 20 个 Python 文件的项目架构

不用子 Agent（主上下文承受全部压力）：
  20 次 read_file 调用，平均每次返回 500 tokens
  工具调用开销：20 * 100 = 2,000 tokens
  工具结果开销：20 * 500 = 10,000 tokens
  中间推理文本：约 3,000 tokens
  ──────────────────────────────
  主上下文增长：~15,000 tokens

使用子 Agent（主上下文保持清洁）：
  task 调用：~150 tokens（prompt + description）
  子 Agent 内部：消耗约 15,000 tokens（但全部被丢弃）
  子 Agent 返回摘要：~500 tokens
  ──────────────────────────────
  主上下文增长：~650 tokens
  节省率：(15000 - 650) / 15000 = 95.7%

什么时候该用：项目文件数 > 5 且分析结果只需要摘要时
```

### 场景 2：多文件批量搜索与替换

```
任务：将 10 个文件中的某个废弃 API 调用替换为新 API

不用子 Agent：
  10 次 read_file（确认需要修改的位置）：5,000 tokens
  10 次 edit_file（执行替换）：2,000 tokens
  中间推理和结果确认：3,000 tokens
  ──────────────────────────────
  主上下文增长：~10,000 tokens

使用子 Agent：
  task 调用（含详细替换指令）：~200 tokens
  子 Agent 内部消耗：~10,000 tokens（被丢弃）
  子 Agent 返回修改报告：~300 tokens
  ──────────────────────────────
  主上下文增长：~500 tokens
  节省率：(10000 - 500) / 10000 = 95%

额外收益：主 Agent 不需要看到每个文件的具体 diff，
只需知道"10 个文件已修改，其中 2 个需要手动验证"
```

### 场景 3：依赖关系审计

```
任务：扫描 30 个包的依赖关系，检查是否存在安全漏洞

不用子 Agent：
  30 次文件读取 + 分析推理
  主上下文增长：~25,000 tokens
  且大量中间细节（每个包的版本号、依赖树）污染后续对话

使用子 Agent：
  task 调用：~200 tokens
  子 Agent 返回审计报告：~800 tokens
  ──────────────────────────────
  主上下文增长：~1,000 tokens
  主 Agent 拿到的是结构化报告：
  "发现 3 个高风险依赖：package-a@1.2.0（CVE-2024-xxxx）、..."

什么时候该用：扫描类任务，结果需要结构化汇总
```

### 决策 Checklist：你的操作适合子 Agent 吗？

在决定是否使用子 Agent 之前，依次回答以下 5 个问题：

**1. 操作是否涉及 3 个以上的文件读取？**
- 是 -> 适合子 Agent。每个文件的内容都会消耗主上下文 token。
- 否 -> 直接执行。1-2 个文件的读取开销可以忽略。

**2. 中间过程对后续对话是否有价值？**
- 否 -> 适合子 Agent。中间的文件内容、命令输出在任务完成后通常不再需要。
- 是 -> 谨慎使用。如果后续步骤需要引用中间结果的具体内容，子 Agent 的摘要可能会遗漏细节。

**3. 结果是否可以浓缩为一段摘要？**
- 是 -> 适合子 Agent。架构分析、代码审计、搜索结果都天然适合摘要。
- 否 -> 不适合。如果用户需要看到每一步的详细输出（如实时日志），子 Agent 会丢失信息。

**4. 任务是否可以独立完成，不需要与用户交互？**
- 是 -> 适合子 Agent。子 Agent 无法向用户提问，只能自主完成任务。
- 否 -> 不适合。如果任务中途需要用户确认或补充信息，必须在主上下文中执行。

**5. 是否有递归派发的风险？**
- 无 -> 安全。子 Agent 的 CHILD_TOOLS 不包含 task 工具，结构上阻止递归。
- 如果你的架构允许子 Agent 创建子 Agent -> 需要额外的递归防护机制。

**快速判断规则**：5 个问题中 3 个或以上为"是/适合"，就使用子 Agent。

---

## 9. 小结

### 9.1 核心要点

| 要点 | 说明 |
|------|------|
| **上下文隔离** | 子 Agent 从 `sub_messages = [{"role": "user", "content": prompt}]` 开始，不继承父历史 |
| **结果摘要** | 用 `"".join(b.text for b in response.content if hasattr(b, "text"))` 提取最终文本 |
| **工具隔离** | `CHILD_TOOLS` 只有 4 个基础工具，`PARENT_TOOLS = CHILD_TOOLS + [task]` |
| **循环上限** | `range(30)` 硬编码，防止无限循环 |
| **Token 效率** | 子 Agent 的所有中间细节被丢弃，主上下文只接收摘要 |
| **调度方式** | task 工具在 `agent_loop` 中通过条件分支处理，不走 `TOOL_HANDLERS` |

### 9.2 关键洞察

```
子 Agent 的核心价值：
1. 保持主上下文的清洁
2. 提供独立的推理空间
3. 实现任务的分解和组合
4. 大幅提高 Token 利用效率（可达 96% 的节省）

设计原则：
- 子 Agent 是"用完即弃"的
- 它的整个 sub_messages 历史都会被丢弃
- 只把最后一次响应中的文本摘要带回父 Agent
- 通过工具隔离（CHILD_TOOLS 不含 task）防止递归

核心洞察：
- "Process isolation gives context isolation for free."
- 上下文隔离是通过创建新的 messages 列表实现的
- 无需任何额外的隔离机制
```

---

## 10. 练习

### 练习 1：异步子 Agent

修改 `run_subagent`，使其可以并行运行多个子 Agent（例如使用 `asyncio` 或 `threading`），并收集所有结果。

### 练习 2：子 Agent 超时

在 `run_subagent` 中添加超时机制，如果子 Agent 运行时间过长（例如超过 60 秒），强制终止并返回部分结果。

### 练习 3：分层摘要

实现多层子 Agent：
- 子 Agent 1 分析文件
- 子 Agent 2 汇总子 Agent 1 的结果
- 主 Agent 使用子 Agent 2 的摘要

注意：这不是递归派发（子 Agent 不能创建子 Agent），而是在父 Agent 中顺序调用多个子 Agent。

### 练习 4：子 Agent 日志

将子 Agent 的完整 `sub_messages` 历史保存到日志文件，便于调试。可以记录每轮的工具调用、结果和 LLM 响应。

### 练习 5：动态轮次限制

将硬编码的 `range(30)` 改为根据 prompt 复杂度动态调整的最大轮次。例如，基于 prompt 的长度、关键词数量等因素来决定。

### 练习 6：组合 Todo 和子 Agent

将 s03 的 TodoManager 和 s04 的子 Agent 组合起来：主 Agent 规划任务（使用 todo），每个任务委托给子 Agent 执行。

---

## 11. 下一步

现在 Agent 可以将复杂任务委托给子 Agent，保持主上下文清洁。但如果每个子任务都重新加载相同的背景知识（如项目规范、编码标准），会很浪费。

在 [s05：技能系统](./s05-skill-loading.md) 中，我们将学习：
- 如何按需加载知识
- 两层注入机制
- SKILL.md 的设计模式

---

< 上一章：[s03-任务规划](./s03-todo-write.md) | [目录](./index.md) | 下一章：[s05-技能系统](./s05-skill-loading.md) >
