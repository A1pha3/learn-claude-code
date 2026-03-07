# s04：子 Agent —— 上下文隔离的艺术

> **核心口号**：*"大任务拆解，每个子任务获得干净上下文"*

> **学习目标**：理解上下文隔离的必要性，掌握子 Agent 模式，学会保持主对话的清洁

---

## 学习目标

完成本章后，你将能够：

1. **理解上下文污染问题** —— 为什么长对话会"变笨"
2. **掌握子 Agent 模式** —— 委托子任务给独立 Agent
3. **理解消息隔离** —— 父子 Agent 的独立上下文
4. **学会结果摘要** —— 将复杂过程浓缩为简洁结论

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

消耗：~3000 tokens
主上下文：增加了 6 条工具调用 + 6 条结果
```

**问题**：为了回答一个简单问题，主上下文被大量中间细节污染。

### 1.2 污染的后果

```
主 Agent 的上下文窗口：

┌─────────────────────────────────────────┐
│ [系统提示] ████████                     │ ← 重要但被淹没
│ [用户问题] ████████                     │
│ [工具调用 1] read_file("setup.py")      │
│ [工具结果 1] ...1000 tokens...          │ ← 大量细节
│ [工具调用 2] read_file("pyproject.toml")│
│ [工具结果 2] ...500 tokens...           │
│ [工具调用 3] read_file("pytest.ini")    │
│ [工具结果 3] ...200 tokens...           │
│ [工具调用 4] ...                        │
│ [工具结果 4] ...                        │
│ [工具调用 5] ...                        │
│ [工具结果 5] ...                        │
│ [工具调用 6] ...                        │
│ [工具结果 6] ...200 tokens...           │
│ [LLM 结论] 项目使用 pytest              │ ← 真正需要的
└─────────────────────────────────────────┘

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
    └─> 子 Agent：
        1. 读取 setup.py
        2. 读取 pyproject.toml
        3. 搜索测试文件
        4. 分析并总结
        └─> 返回："项目使用 pytest，配置在 pytest.ini 中"

  主 Agent：收到简洁的答案（~50 tokens）
  上下文保持清洁
```

**核心思想**：子 Agent 承担"脏活累活"，主 Agent 只看结论。

---

## 2. 子 Agent 模式

### 2.1 架构对比

```
没有子 Agent（单 Agent）：
┌─────────────────────────────────────┐
│         主 Agent                    │
│                                     │
│  1. read_file("a.py")  → 大量输出   │
│  2. read_file("b.py")  → 大量输出   │
│  3. read_file("c.py")  → 大量输出   │
│  4. read_file("d.py")  → 大量输出   │
│  5. read_file("e.py")  → 大量输出   │
│  6. 总结                          │
│                                     │
│  所有细节都在主上下文中              │
└─────────────────────────────────────┘

有子 Agent（父子模式）：
┌─────────────────┐
│    主 Agent     │
│                 │
│  1. task("分析...") │ ────────┐
│  2. 收到摘要      │         │
│                  │         ▼
│  只看到结论      │    ┌──────────┐
│                  │    │ 子 Agent │
└──────────────────┘    │          │
                       │ 1. 读取 5 个文件
                       │ 2. 分析
                       │ 3. 返回摘要
                       └──────────┘
```

### 2.2 父子 Agent 的工具分配

```python
# 父 Agent 的工具（包含 task）
PARENT_TOOLS = [
    {"name": "read_file", ...},
    {"name": "write_file", ...},
    {"name": "run_bash", ...},
    {"name": "edit_file", ...},
    {"name": "todo", ...},
    {"name": "task", ...},  # ← 新增：创建子 Agent
]

# 子 Agent 的工具（不包含 task，防止递归）
CHILD_TOOLS = [
    {"name": "read_file", ...},
    {"name": "write_file", ...},
    {"name": "run_bash", ...},
    {"name": "edit_file", ...},
    {"name": "todo", ...},
    # 没有 task 工具
]
```

**为什么子 Agent 不能有 task 工具？**

防止无限递归：子 Agent 创建子子 Agent，子子 Agent 创建子子子 Agent...

### 2.3 task 工具定义

```python
TOOLS.append({
    "name": "task",
    "description": "创建一个子 Agent 来处理独立的子任务。子 Agent 有干净的上"
                   "下文，完成后返回摘要。适用于需要多个文件读取或复杂分析的"
                   "任务。",
    "input_schema": {
        "type": "object",
        "properties": {
            "prompt": {
                "type": "string",
                "description": "给子 Agent 的详细指令"
            }
        },
        "required": ["prompt"]
    }
})
```

---

## 3. 子 Agent 实现

### 3.1 核心函数

```python
def run_subagent(prompt: str) -> str:
    """运行子 Agent 并返回摘要"""
    # 子 Agent 从干净的消息列表开始
    sub_messages = [{"role": "user", "content": prompt}]

    # 限制子 Agent 的最大轮次
    MAX_SUB_ROUNDS = 30

    for _ in range(MAX_SUB_ROUNDS):
        response = client.messages.create(
            model=MODEL,
            system=SUBAGENT_SYSTEM,  # 可能与主 Agent 不同
            messages=sub_messages,
            tools=CHILD_TOOLS,        # 子 Agent 的工具集
            max_tokens=8000,
        )

        sub_messages.append({
            "role": "assistant",
            "content": response.content
        })

        # 检查是否继续
        if response.stop_reason != "tool_use":
            break  # 子 Agent 完成工作

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
                        "content": str(output)[:50000],
                    })

        sub_messages.append({
            "role": "user",
            "content": results
        })

    # 提取最终文本作为摘要
    summary_blocks = [
        block.text for block in response.content
        if hasattr(block, 'text')
    ]

    return "".join(summary_blocks) or "(无摘要)"
```

### 3.2 关键设计点

**1. 干净的开始**

```python
sub_messages = [{"role": "user", "content": prompt}]
# 不是：sub_messages = parent_messages.copy()
```

子 Agent 不继承父 Agent 的历史，从零开始。

**2. 独立的系统提示**

```python
SUBAGENT_SYSTEM = """你是子 Agent，负责执行特定的子任务。
- 专注于完成给定的任务
- 完成后返回简洁的摘要
- 不要与用户交互，只返回结果
"""
```

子 Agent 可以有与主 Agent 不同的角色和行为。

**3. 轮次限制**

```python
MAX_SUB_ROUNDS = 30  # 防止无限循环
```

保护机制：即使子 Agent 出问题，也不会无限运行。

**4. 结果提取**

```python
summary_blocks = [
    block.text for block in response.content
    if hasattr(block, 'text')
]
return "".join(summary_blocks)
```

只返回最终的文本摘要，丢弃所有中间的工具调用和结果。

---

## 4. 完整示例

### 4.1 用户请求

```
用户：帮我分析这个项目的架构，找出所有模块及其职责
```

### 4.2 主 Agent 的处理

```
轮次 1（主 Agent）：
  用户：分析项目架构...
  主 Agent：这个任务需要读取多个文件，让我用子 Agent
        tool_use: task(prompt="分析 /workspace 目录下的 Python 模块，
                       找出所有 .py 文件，分析它们的职责和依赖关系")

  [子 Agent 开始运行...]
```

### 4.3 子 Agent 的处理

```
子 Agent（独立上下文）：
  prompt：分析 /workspace 目录下的 Python 模块...

  子 Agent 轮次 1：
    tool_use: run_bash("find . -name '*.py' -type f")
    结果：[20 个文件的列表]

  子 Agent 轮次 2-10：
    tool_use: read_file("auth.py")
    tool_use: read_file("database.py")
    tool_use: read_file("api.py")
    ... 读取并分析所有文件

  子 Agent 轮次 11：
    stop_reason = "end_turn"
    返回摘要：
      "项目包含以下模块：
      1. auth.py - 用户认证和授权
      2. database.py - 数据库连接和操作
      3. api.py - REST API 端点
      4. utils.py - 通用工具函数
      ...
      模块依赖：api → auth → database"
```

### 4.4 主 Agent 收到摘要

```
轮次 1（主 Agent）：
  tool_result：[子 Agent 的摘要]
  结果："项目包含以下模块：1. auth.py - 用户认证..."

轮次 2（主 Agent）：
  主 Agent：根据子 Agent 的分析，项目架构如下：
    [简洁地展示结果]

  上下文保持清洁：只有 2 轮，没有 20 个文件读取的细节
```

---

## 5. 上下文隔离的价值

### 5.1 Token 消耗对比

```
没有子 Agent：
  读取 20 个文件 × 平均 500 tokens = 10,000 tokens
  加上工具调用和中间处理 = ~15,000 tokens

有子 Agent：
  task 调用 = ~100 tokens
  子 Agent 摘要 = ~500 tokens
  主 Agent 只消耗 = ~600 tokens

节省：~14,400 tokens（96%）
```

### 5.2 系统提示的保持

```
没有子 Agent：
  系统提示：[1000 tokens]
  工具结果：[15000 tokens]
  ────────────────────────────
  系统提示占比：1000/16000 = 6%（可能被忽略）

有子 Agent：
  系统提示：[1000 tokens]
  子 Agent 摘要：[500 tokens]
  ────────────────────────────
  系统提示占比：1000/1500 = 67%（保持有效）
```

### 5.3 可组合性

子 Agent 使任务可以分解和组合：

```
主任务：重构整个项目
├── 子任务 1：分析当前架构（子 Agent A）
├── 子任务 2：设计新架构（子 Agent B）
├── 子任务 3：迁移代码（子 Agent C）
│   ├── 子子任务 3.1：迁移模块 A（子 Agent C1）
│   ├── 子子任务 3.2：迁移模块 B（子 Agent C2）
│   └── 子子任务 3.3：迁移模块 C（子 Agent C3）
└── 子任务 4：验证迁移（子 Agent D）
```

每个子任务独立执行，只返回摘要给父任务。

---

## 6. 与 s03 的对比

| 方面 | s03 | s04 |
|------|-----|-----|
| 规划 | TodoManager（内存） | 子 Agent（独立上下文） |
| 上下文 | 单一共享 | 父子隔离 |
| 工具数量 | 5 | 5 (子) + 1 (父: task) |
| 结果返回 | 完整历史 | 摘要文本 |
| Token 效率 | 中等 | 高 |

---

## 7. 常见问题

### Q1：子 Agent 和工具有什么区别？

| 特性 | 工具 | 子 Agent |
|------|------|----------|
| 能力 | 单一操作 | 多步骤推理 |
| 上下文 | 共享 | 独立 |
| 返回 | 原始结果 | 摘要 |
| 示例 | read_file | task("分析所有测试文件") |

**答案**：子 Agent 是"有 LLM 的工具"，可以执行需要多步推理的任务。

### Q2：子 Agent 可以访问文件吗？

**答案**：可以。子 Agent 继承了 `read_file`、`write_file` 等文件操作工具，只是在独立上下文中运行。

### Q3：如何控制子 Agent 的行为？

**答案**：通过 `SUBAGENT_SYSTEM` 系统提示：

```python
# 严格模式
SUBAGENT_SYSTEM = """你只执行任务，不提出问题。完成后返回摘要。"""

# 交互模式
SUBAGENT_SYSTEM = """你可以提出澄清问题，但最终必须返回摘要。"""

# 调试模式
SUBAGENT_SYSTEM = """详细记录你的思考过程，返回完整的推理链。"""
```

### Q4：子 Agent 失败了怎么办？

```python
try:
    summary = run_subagent(prompt)
except Exception as e:
    summary = f"子任务执行失败: {str(e)}"
# 将错误作为摘要返回，主 Agent 可以决定如何处理
```

---

## 8. 最佳实践

### 8.1 何时使用子 Agent

```
✅ 适合使用子 Agent：
- 需要读取多个文件（3+）
- 复杂的分析任务
- 需要多步推理的子任务
- 想要保持主上下文清洁

❌ 不需要子 Agent：
- 简单的单步操作
- 快速文件读取（1-2 个文件）
- 直接的工具调用就足够
```

### 8.2 编写好的子任务提示

```python
# ❌ 模糊的提示
task("分析代码")

# ✅ 明确的提示
task("""
分析 /workspace 目录的 Python 代码结构：
1. 找出所有 .py 文件
2. 分析每个文件的主要类和函数
3. 识别模块之间的依赖关系
4. 返回简洁的架构摘要
""")
```

### 8.3 子 Agent 的轮次控制

```python
# 简单任务：少轮次
MAX_SUB_ROUNDS_SIMPLE = 10

# 复杂任务：多轮次
MAX_SUB_ROUNDS_COMPLEX = 50

# 动态调整（根据提示长度）
max_rounds = min(30, 10 + len(prompt) // 100)
```

---

## 9. 小结

### 9.1 核心要点

| 要点 | 说明 |
|------|------|
| **上下文隔离** | 子 Agent 从干净的状态开始 |
| **结果摘要** | 只返回最终结论，丢弃中间细节 |
| **工具分离** | 子 Agent 不继承 task 工具 |
| **Token 效率** | 大幅减少主上下文的消耗 |

### 9.2 代码模板

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]

    for _ in range(MAX_SUB_ROUNDS):
        response = client.messages.create(
            messages=sub_messages,
            tools=CHILD_TOOLS,
        )
        sub_messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            break

        # 执行工具，收集结果...

    # 返回摘要
    return extract_summary(response)

# 注册
TOOL_HANDLERS["task"] = lambda **kw: run_subagent(kw["prompt"])
```

### 9.3 关键洞察

```
子 Agent 的核心价值：
1. 保持主上下文的清洁
2. 提供独立的推理空间
3. 实现任务的分解和组合
4. 大幅提高 Token 利用效率

设计原则：
- 子 Agent 是"用完即弃"的
- 它的整个历史都会被丢弃
- 只把摘要带回父 Agent
```

---

## 10. 练习

### 练习 1：异步子 Agent

修改 `run_subagent`，使其可以并行运行多个子 Agent，并收集所有结果。

### 练习 2：子 Agent 超时

添加超时机制，如果子 Agent 运行时间过长，强制终止并返回部分结果。

### 练习 3：分层摘要

实现多层子 Agent：
- 子 Agent 1 分析文件
- 子 Agent 2 汇总子 Agent 1 的结果
- 主 Agent 使用子 Agent 2 的摘要

### 练习 4：子 Agent 日志

将子 Agent 的完整消息历史保存到日志文件，便于调试。

---

## 11. 下一步

现在 Agent 可以将复杂任务委托给子 Agent，保持主上下文清洁。但如果每个子任务都重新加载相同的背景知识（如项目规范、编码标准），会很浪费。

在 [s05：技能系统](./s05-skill-loading.md) 中，我们将学习：
- 如何按需加载知识
- 两层注入机制
- SKILL.md 的设计模式

---

< 上一章：[s03-任务规划](./s03-todo-write.md) | [目录](./index.md) | 下一章：[s05-技能系统](./s05-skill-loading.md) >
