# s01：Agent 循环 —— 最小可用 Agent

> **核心口号**：*"一个循环 + 一个工具 = Agent"*

> **学习目标**：理解 Agent 的本质，用不到 30 行代码实现第一个可工作的 AI 编程助手

---

## 学习目标

完成本章后，你将能够：

1. **理解 Agent 的核心机制** —— 消息循环的本质
2. **实现最小可用 Agent** —— 用不到 30 行 Python 代码
3. **掌握 `stop_reason` 的作用** —— 理解循环控制的关键
4. **理解消息累积模式** —— 为什么需要消息列表

---

## 0. 上手演练（建议先做）

先跑一遍最小 Agent，再读原理细节：

```bash
uv run python agents/s01_agent_loop.py
```

建议重点观察：

1. `stop_reason` 何时从 `tool_use` 变为结束态。
2. 工具结果是如何回到消息列表并影响下一轮决策的。
3. 循环结构为什么几乎不需要随功能扩展而改变。

---

## 1. 问题的本质

### 1.1 LLM 的困境

想象你是一个被困在房间里绝世聪明的顾问：

```
┌─────────────────────┐
│                     │
│   🧠 LLM（你）      │
│                     │
│   "我想知道         │
│    requirements.txt │
│    的内容..."       │
│                     │
└─────────────────────┘

🚫 没有手
🚫 没有眼睛
🚫 没有耳朵
🚫 无法接触外部世界
```

你可以思考，可以给出建议，但你无法：
- 打开文件读取内容
- 运行代码查看结果
- 检查错误日志
- 执行任何需要"手"的操作

**这就是 LLM 的根本局限**：它被困在文本世界里。

### 1.2 传统做法的问题

在没有 Agent 之前，用户需要手动充当"循环"：

```
用户：帮我看看 requirements.txt 里有什么依赖
LLM：请运行 cat requirements.txt
用户：[手动运行命令] anthropic==0.18.0
LLM：我看到你使用 anthropic 0.18.0。现在你想做什么？
用户：帮我安装这些依赖
LLM：请运行 pip install -r requirements.txt
用户：[手动运行命令] ...
```

**问题**：每一步都需要用户手动传递结果，效率低下。

### 1.3 Agent 的解决方案

Agent 让 LLM 自己完成这个循环：

```
用户：帮我看看 requirements.txt 里有什么依赖并安装
Agent：
  1. 调用 read_file("requirements.txt")
  2. 获得结果：anthropic==0.18.0
  3. 调用 run_bash("pip install -r requirements.txt")
  4. 获得结果：Successfully installed
  5. 返回给用户：依赖已安装完成
```

**关键**：Agent 自动处理"调用 → 获取结果 → 继续思考"的循环，无需用户介入。

---

## 2. 核心设计

### 2.1 架构图

```
用户输入
   │
   ▼
┌─────────┐
│ 消息列表 │ ◄──────┐
│ messages│        │
└────┬────┘        │
     │             │
     ▼             │
┌─────────┐        │
│   LLM   │        │ add messages
└────┬────┘        │
     │             │
     ▼             │
┌─────────┐        │
│ stop_reason? │    │
└────┬────┘        │
     │             │
     ├─否─→ 返回结果──┤
     │             │
     ├─是─────────┘
     │
     ▼
┌─────────┐
│ 执行工具 │
└────┬────┘
     │
     ▼
┌─────────┐
│ 收集结果 │
└────┬────┘
     │
     ▼
add to messages → 回到 LLM
```

### 2.2 数据结构

**消息列表**：整个对话历史的容器

```python
messages = [
    # 轮次 1
    {"role": "user", "content": "用户的问题"},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "tool_1", "name": "bash", "input": {...}}
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "tool_1", "content": "命令输出"}
    ]},

    # 轮次 2
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "tool_2", "name": "bash", "input": {...}}
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "tool_2", "content": "命令输出"}
    ]},

    # ... 可能有很多轮
]
```

**关键点**：每次调用 LLM，都发送完整的 `messages` 列表。这确保 LLM 能看到完整的历史。

---

## 3. 代码实现

### 3.1 完整代码（不到 30 行）

```python
import os
from anthropic import Anthropic

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
MODEL = "claude-3-5-sonnet-20241022"

def agent_loop(query):
    # 1. 初始化消息列表
    messages = [{"role": "user", "content": query}]

    # 2. 循环直到 LLM 不再调用工具
    while True:
        response = client.messages.create(
            model=MODEL,
            system="You are a helpful coding assistant.",
            messages=messages,
            tools=[{  # 只有一个工具：bash
                "name": "run_bash",
                "description": "Execute a bash command",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "command": {"type": "string"}
                    },
                    "required": ["command"]
                }
            }],
            max_tokens=8000,
        )

        # 3. 添加 LLM 响应到消息列表
        messages.append({"role": "assistant", "content": response.content})

        # 4. 检查是否继续循环
        if response.stop_reason != "tool_use":
            break  # LLM 完成工作，退出循环

        # 5. 执行工具并收集结果
        results = []
        for block in response.content:
            if block.type == "tool_use":
                command = block.input["command"]
                output = os.popen(command).read()[:50000]
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output
                })

        # 6. 将结果添加到消息列表，继续下一轮
        messages.append({"role": "user", "content": results})

    # 7. 返回最终响应
    return response.content

# 运行
if __name__ == "__main__":
    result = agent_loop("列出当前目录的 Python 文件")
    print(result)
```

### 3.2 代码逐行解析

```python
# ========== 第 1 部分：初始化 ==========
messages = [{"role": "user", "content": query}]
```
- 创建消息列表，初始只有用户的提问
- 这是整个对话的起点

```python
# ========== 第 2 部分：LLM 调用 ==========
response = client.messages.create(
    model=MODEL,
    system="You are a helpful coding assistant.",
    messages=messages,  # 完整的历史消息
    tools=[...],
    max_tokens=8000,
)
```
- 发送完整历史给 LLM
- `tools` 参数告诉 LLM 有哪些可用工具

```python
# ========== 第 3 部分：保存响应 ==========
messages.append({"role": "assistant", "content": response.content})
```
- 将 LLM 的响应添加到历史
- 这个响应可能包含 `tool_use` 类型的块

```python
# ========== 第 4 部分：检查循环条件 ==========
if response.stop_reason != "tool_use":
    break
```
- `stop_reason` 告诉我们 LLM 为什么停止
- 只有 `tool_use` 时才继续循环

```python
# ========== 第 5 部分：执行工具 ==========
for block in response.content:
    if block.type == "tool_use":
        command = block.input["command"]
        output = os.popen(command).read()[:50000]
        results.append({...})
```
- 遍历 LLM 响应中的每个块
- 找到 `tool_use` 类型的块
- 执行命令并收集结果

```python
# ========== 第 6 部分：添加结果并继续 ==========
messages.append({"role": "user", "content": results})
```
- 将工具结果添加到消息列表
- 作为"用户"角色，因为这是系统反馈
- 循环回到第 2 部分，继续下一轮

---

## 4. 关键概念详解

### 4.1 stop_reason 的作用

`stop_reason` 是循环控制的关键：

| stop_reason | 含义 | 行动 |
|-------------|------|------|
| `tool_use` | LLM 想要调用工具 | 执行工具，继续循环 |
| `end_turn` | LLM 完成了响应 | 退出循环，返回结果 |
| `max_tokens` | 达到 Token 上限 | 需要处理（通常是错误） |

**为什么叫 `tool_use` 而不是 `call_tool`？**

因为 LLM 的响应"使用"了工具，但不一定是"调用"工具。它只是表达了调用意图，真正的执行由外部代码完成。

### 4.2 消息列表的角色

消息列表是 Agent 的**短期记忆**：

```
轮次 1：
messages = [用户问题]

轮次 2：
messages = [用户问题, LLM响应(含工具调用), 工具结果]

轮次 3：
messages = [用户问题, LLM响应1, 工具结果1, LLM响应2, 工具结果2]

...
```

**为什么需要完整历史？**

LLM 是无状态的，每次调用都是独立的。要让它"记住"之前的对话，必须在每次调用时发送完整历史。

### 4.3 工具结果的格式

工具结果必须匹配对应的工具调用：

```python
{
    "type": "tool_result",      # 必须是 tool_result
    "tool_use_id": block.id,    # 必须匹配原工具调用的 id
    "content": "命令的输出"      # 实际的结果内容
}
```

**`tool_use_id` 的作用**：让 LLM 知道这个结果对应哪个工具调用。当一次 LLM 响应包含多个工具调用时，这非常重要。

---

## 5. 运行示例

### 5.1 示例 1：列出文件

**用户输入**：
```
列出当前目录的 Python 文件
```

**执行流程**：

```
轮次 1：
  LLM：我需要运行 ls 命令
        tool_use: run_bash("ls *.py")
  系统：执行 ls *.py
        结果：s01_agent_loop.py
              s02_tool_use.py
              ...

轮次 2：
  LLM：我看到了这些文件：
        s01_agent_loop.py
        s02_tool_use.py
        ...
  系统：stop_reason = "end_turn"
  返回
```

### 5.2 示例 2：多步骤任务

**用户输入**：
```
创建一个文件 hello.py，内容是 print("Hello")，然后运行它
```

**执行流程**：

```
轮次 1：
  LLM：创建文件
        tool_use: run_bash("echo 'print(\"Hello\")' > hello.py")

轮次 2：
  LLM：现在运行它
        tool_use: run_bash("python hello.py")

轮次 3：
  LLM：程序已运行，输出是 "Hello"
  返回
```

**关键点**：Agent 自动处理了多个步骤，用户只说了一次话。

---

## 6. 常见问题

### Q1：为什么工具结果作为"用户"角色？

```python
messages.append({"role": "user", "content": results})  # 为什么是 user？
```

**答案**：在多轮对话中，任何非 assistant 的内容都被视为"用户输入"。工具结果是系统反馈，对 LLM 来说就像是用户在提供信息。

### Q2：如果工具执行失败怎么办？

```python
output = os.popen(command).read()[:50000]
# 如果命令失败，output 会包含错误信息
```

**答案**：将错误信息作为 `content` 返回给 LLM，让它决定如何处理。这保持了 Agent 的自主性。

### Q3：为什么要限制输出长度？

```python
output = os.popen(command).read()[:50000]  # 只取前 50000 字符
```

**答案**：
1. 防止单个工具结果消耗过多上下文
2. 大多数情况下，50000 字符足够了
3. 如果需要更多，LLM 可以要求读取特定部分

### Q4：循环最多执行多少次？

```python
while True:  # 无限循环？
```

**答案**：实际上不是无限的：
1. LLM 最终会停止调用工具（`stop_reason != "tool_use"`）
2. 可以添加最大循环次数限制（生产环境建议）

改进版本：
```python
MAX_ROUNDS = 50
for _ in range(MAX_ROUNDS):
    # ... 循环逻辑
    if response.stop_reason != "tool_use":
        break
```

---

## 7. 调试技巧

### 7.1 打印消息历史

```python
def debug_print(messages):
    for i, msg in enumerate(messages):
        print(f"[{i}] {msg['role']}: {msg['content'][:100]}...")
```

### 7.2 打印 stop_reason

```python
print(f"stop_reason: {response.stop_reason}")
```

### 7.3 打印工具调用详情

```python
for block in response.content:
    if block.type == "tool_use":
        print(f"Calling: {block.name}({block.input})")
```

---

## 8. 与后续章节的关系

### 8.1 这个循环永不改变

```
s01: while stop_reason == "tool_use": ...
s02: while stop_reason == "tool_use": ...  # 相同
s03: while stop_reason == "tool_use": ...  # 相同
...
s12: while stop_reason == "tool_use": ...  # 仍然相同
```

**核心洞察**：所有后续章节都是在循环"之前"和"之后"添加逻辑，而不是改变循环本身。

### 8.2 s02 会添加什么

s02 将添加更多工具，但循环结构保持不变：

```python
# s01: 只有 bash
TOOL_HANDLERS = {"bash": run_bash}

# s02: 有多个工具
TOOL_HANDLERS = {
    "bash": run_bash,
    "read_file": run_read,
    "write_file": run_write,
    "edit_file": run_edit,
}
# 但循环逻辑完全相同！
```

---

## 9. 小结

### 9.1 核心要点

| 要点 | 说明 |
|------|------|
| **Agent 的本质** | LLM + 工具 + 循环 |
| **循环控制** | `stop_reason == "tool_use"` 时继续 |
| **消息列表** | 完整的对话历史，每次调用都发送 |
| **工具结果** | 作为"用户"角色添加回消息列表 |

### 9.2 代码模板

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]

    while True:
        response = client.messages.create(messages=messages, tools=TOOLS)
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            break

        results = execute_tools(response.content)
        messages.append({"role": "user", "content": results})

    return response.content
```

**这个模板贯穿整个课程**，记住它！

### 9.3 关键洞察

```
整个 Agent 就是一个 while 循环。

理解了这个循环，你就理解了 Agent 的一半。

另一半是：如何设计好的工具。
```

---

## 10. 练习

### 练习 1：理解循环

修改代码，在每次循环开始时打印当前轮次和消息数量。

### 练习 2：添加限制

添加 `MAX_ROUNDS` 常量，限制最大循环次数。

### 练习 3：错误处理

如果 `os.popen(command)` 执行失败（如命令不存在），返回友好的错误信息。

### 练习 4：多工具并行

当 LLM 在一次响应中调用多个工具时，验证所有工具都被正确执行。

---

## 11. 下一步

你已经实现了第一个 Agent！虽然它只有一个 `bash` 工具，但核心结构已经完整。

在 [s02：工具系统](./s02-tool-use.md) 中，我们将学习：
- 如何添加多个工具
- 如何实现安全的文件操作
- 如何使用分发器简化代码

---

< 上一章：[00-基础篇](./00-basics.md) | [目录](./index.md) | 下一章：[s02-工具系统](./s02-tool-use.md) >
