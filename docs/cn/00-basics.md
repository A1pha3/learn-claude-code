# 00 基础篇：构建 Agent 的背景知识

> **学习目标**：理解构建 AI Agent 所需的核心概念，为后续实践打下坚实基础

---

## 学习目标

完成本章后，你将理解：

1. **什么是 LLM 以及它的局限性** —— 为什么需要 Agent
2. **工具调用的本质** —— LLM 如何与外部世界交互
3. **上下文窗口的概念** —— 为什么需要压缩和记忆管理
4. **流式 vs 批量处理** —— 两种交互模式的权衡
5. **Agent 的核心模式** —— 理解「消息循环」的设计思想

---

## 0. 上手演练（建议先做）

先运行基础章节对应脚本，再阅读正文，会更容易建立整体心智模型：

```bash
uv run python agents/s01_agent_loop.py
```

建议带着这 3 个问题阅读后续内容：

1. LLM 自身能做什么，不能做什么？
2. 为什么“工具调用”是 Agent 必需能力？
3. 为什么必须用循环把“思考—执行—反馈”串起来？

### 本章完成标准

- 能用自己的话解释“LLM”与“Agent”在能力边界上的区别。
- 能画出最小工具调用闭环并说明每一步职责。
- 能把本章概念映射到后续章节的具体机制。

---

## 1. 什么是 LLM（大语言模型）

### 1.1 LLM 能做什么

大语言模型（Large Language Model）是一个**文本补全引擎**：

```
输入： "法国的首都是"
输出： "巴黎"
```

但它可以做的事情远不止简单的问答：

| 能力 | 说明 | 示例 |
|------|------|------|
| **理解** | 理解自然语言指令 | "帮我写一个冒泡排序" |
| **推理** | 进行逻辑推导 | "如果 A > B 且 B > C，那么 A 和 C 的关系是？" |
| **生成** | 创造新内容 | "写一首关于春天的诗" |
| **转换** | 改变内容形式 | "把这段代码从 Python 改成 JavaScript" |
| **总结** | 提取关键信息 | "用三句话总结这篇文章" |

### 1.2 LLM 的局限

LLM 有一个根本性的局限：**它只能处理文本，不能直接操作外部世界**。

```
❌ LLM 不能直接做的事情：
   - 读取文件系统中的文件
   - 运行代码或命令
   - 访问网络 API
   - 查看数据库
   - 执行任何非文本操作

✅ LLM 能做的事情：
   - 理解你的意图
   - 决定需要执行什么操作
   - 生成操作的指令
   - 解释操作的结果
```

**类比**：LLM 就像一个被困在房间里的专家顾问。他可以给你最好的建议，但无法帮你去倒一杯水。你需要成为他的"手"，执行他建议的操作。

---

## 2. 工具调用（Tool Use）

### 2.1 什么是工具调用

工具调用是 LLM 与外部世界交互的**标准化协议**。它让 LLM 能够：

```
1. 表达"我想调用某个工具"
2. 指定工具的参数
3. 接收工具的执行结果
4. 基于结果继续思考
```

### 2.2 工具调用的格式

一个典型的工具调用包含三个部分：

```json
// 1. LLM 的响应（包含工具调用）
{
  "content": [
    {
      "type": "tool_use",
      "id": "tool_123abc",
      "name": "read_file",
      "input": {
        "path": "requirements.txt"
      }
    }
  ]
}

// 2. 系统执行工具后的响应
{
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "tool_123abc",
      "content": "anthropic==0.18.0\npython-dotenv==1.0.0"
    }
  ]
}

// 3. LLM 继续思考
{
  "content": [
    {
      "type": "text",
      "text": "我看到项目使用 anthropic 0.18.0 版本..."
    }
  ]
}
```

### 2.3 工具定义

工具需要提前定义，告诉 LLM 有哪些可用工具：

```python
TOOLS = [
    {
        "name": "read_file",
        "description": "读取文件内容",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "文件路径"
                }
            },
            "required": ["path"]
        }
    },
    {
        "name": "write_file",
        "description": "写入文件",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "content": {"type": "string"}
            },
            "required": ["path", "content"]
        }
    },
    {
        "name": "run_bash",
        "description": "执行 Bash 命令",
        "input_schema": {
            "type": "object",
            "properties": {
                "command": {
                    "type": "string",
                    "description": "要执行的命令"
                }
            },
            "required": ["command"]
        }
    }
]
```

**关键点**：工具定义是 LLM 唯一能看到的"能力清单"。它只能调用已定义的工具。

### 2.4 stop_reason：判断何时停止

每次 LLM 响应都会返回一个 `stop_reason`，告诉我们为什么停止生成：

| stop_reason | 含义 | 下一步操作 |
|-------------|------|------------|
| `tool_use` | LLM 想要调用工具 | 执行工具，返回结果，继续循环 |
| `end_turn` | LLM 完成了响应 | 将结果返回给用户 |
| `max_tokens` | 达到 Token 上限 | 可能需要压缩上下文 |

**核心机制**：`stop_reason == "tool_use"` 是 Agent 循环继续的唯一条件。

---

## 3. 上下文窗口

### 3.1 什么是上下文窗口

上下文窗口是 LLM 一次能处理的**最大文本量**，通常用 Token 数衡量：

| 模型 | 上下文窗口 |
|------|------------|
| Claude 3 Haiku | 200,000 tokens |
| Claude 3 Sonnet | 200,000 tokens |
| Claude 3 Opus | 200,000 tokens |
| Claude 3.5 Sonnet | 200,000 tokens |

### 3.2 Token 的概念

Token 不等于单词或字符。粗略估算：

```
英文：1 token ≈ 4 个字符 ≈ 0.75 个单词
中文：1 token ≈ 1-2 个汉字
代码：1 token ≈ 3-5 个字符（取决于语言）
```

**示例**：

```
文本： "Hello, world!"
Token： Hello | , | world | ! (4 tokens)

文本： "你好，世界！"
Token： 你 | 好 | ， | 世 | 界 | ！ (6 tokens)
```

### 3.3 上下文的构成

每次调用 LLM，需要发送完整的消息历史：

```python
messages = [
    {"role": "user", "content": "用户的问题"},
    {"role": "assistant", "content": [ToolUse(...)]},
    {"role": "user", "content": [ToolResult(...)]},
    {"role": "assistant", "content": [ToolUse(...)]},
    {"role": "user", "content": [ToolResult(...)]},
    # ... 可能有几十轮
]
```

**问题**：随着对话进行，上下文会不断增长，最终可能超出窗口限制。

### 3.4 解决方案：上下文压缩

在第 6 章（s06）中，我们将学习三种压缩策略：

```
层级 1：微压缩
├── 用占位符替换旧的工具结果
└── 例如："[Previous: used read_file]"

层级 2：自动压缩
├── 当 Token 超过阈值时触发
├── 保存完整记录到磁盘
└── 用摘要替换历史消息

层级 3：手动压缩
├── Agent 主动调用 compact 工具
└── 与自动压缩相同的机制
```

---

## 4. 流式 vs 批量处理

### 4.1 批量处理（Blocking）

等待完整响应后再处理：

```python
response = client.messages.create(
    model=MODEL,
    messages=messages,
    max_tokens=1000,
)
# 等待...可能几秒钟
print(response.content)  # 一次性获得完整响应
```

**优点**：简单，易于理解
**缺点**：用户等待时间长，体验差

### 4.2 流式处理（Streaming）

逐块接收响应：

```python
with client.messages.stream(
    model=MODEL,
    messages=messages,
    max_tokens=1000,
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)  # 实时输出
```

**优点**：用户立即看到输出，体验好
**缺点**：实现稍复杂

### 4.3 Agent 中的选择

对于 Agent 构建，**批量处理更常见**，因为：

1. Agent 主要与工具交互，不需要实时显示中间文本
2. 需要检查完整的 `stop_reason` 才能决定下一步
3. 简化了错误处理和重试逻辑

但在面向用户的聊天界面中，**流式处理是必须的**。

---

## 5. Agent 的核心模式

### 5.1 什么是 Agent

Agent = LLM + 工具 + 循环

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   ┌───────┐      ┌───────┐      ┌───────────┐     │
│   │ 用户   │ ---> │  LLM  │ ---> │   工具    │     │
│   │ 提问   │      │       │ <--- │   执行    │     │
│   └───────┘      └───────┘      └───────────┘     │
│                          ↑                          │
│                          │                          │
│                    工具结果                        │
│                          │                          │
│                    (循环继续)                       │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 5.2 消息循环的伪代码

```python
def agent_loop(user_query):
    # 1. 初始化消息列表
    messages = [{"role": "user", "content": user_query}]

    # 2. 循环直到 LLM 不再调用工具
    while True:
        # 2.1 调用 LLM
        response = llm.call(messages=messages, tools=TOOLS)
        messages.append({"role": "assistant", "content": response.content})

        # 2.2 检查是否继续
        if response.stop_reason != "tool_use":
            break  # LLM 完成工作，返回结果

        # 2.3 执行工具并收集结果
        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = execute_tool(block.name, block.input)
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output
                })

        # 2.4 将结果添加到消息，继续循环
        messages.append({"role": "user", "content": results})

    # 3. 返回最终响应
    return messages[-1]["content"]
```

**关键洞察**：整个 Agent 就是这四步的循环。后续 12 个章节的所有机制，都是在这个基础上的增强。

### 5.3 为什么循环如此重要

```
没有循环（单次调用）：
用户 → LLM → 工具调用 → [停止，需要人工介入]
用户 → LLM → 工具调用 → [停止，需要人工介入]
...每次都需要用户手动传递结果

有循环（Agent）：
用户 → LLM → 工具调用 → 结果 → LLM → 工具调用 → 结果 → LLM → ...
[自动持续，直到任务完成]
```

---

## 6. 系统提示（System Prompt）

### 6.1 什么是系统提示

系统提示是给 LLM 的"指令集"，定义它的角色和行为：

```python
SYSTEM = """你是一个专业的编程助手，工作在 /workspace 目录中。

你的职责：
- 帮助用户编写、调试和优化代码
- 遵循项目的代码规范
- 在修改前先理解现有代码结构

可用工具：
- read_file: 读取文件
- write_file: 写入文件
- run_bash: 执行命令

请始终用中文回复。"""
```

### 6.2 系统提示 vs 消息历史

| 方面 | 系统提示 | 消息历史 |
|------|----------|----------|
| 作用 | 定义角色、规则、能力 | 记录对话内容 |
| 变化 | 通常固定 | 每轮增长 |
| Token 占用 | 较少 | 较大（随对话增长） |

**设计原则**：系统提示应该只包含**必须的、不随对话变化的**信息。动态信息（如技能详情）应该按需加载（s05 会讲解）。

---

## 7. 小结与下一步

### 7.1 核心概念回顾

| 概念 | 关键要点 |
|------|----------|
| **LLM** | 文本补全引擎，只能处理文本，不能直接操作外部世界 |
| **工具调用** | LLM 与外部世界交互的标准化协议 |
| **stop_reason** | `tool_use` 表示继续循环，其他表示可以返回 |
| **上下文窗口** | 一次能处理的最大文本量，需要管理 |
| **Agent 循环** | LLM → 工具 → 结果 → LLM 的无限循环 |
| **系统提示** | 定义 Agent 的角色和行为规则 |

### 7.2 关键洞察

```
整个 Agent 的核心就是一个 while 循环：

while stop_reason == "tool_use":
    call_llm()
    execute_tools()
    append_results()

所有其他机制（规划、技能、压缩、团队）都是
在这个循环的"前"和"后"添加逻辑，而不是改变循环本身。
```

### 7.3 下一步

你已经掌握了构建 Agent 所需的基础知识。在下一章 [s01-Agent 循环](./s01-the-agent-loop.md) 中，我们将用不到 30 行 Python 代码实现第一个可工作的 Agent。

---

## 思考题

在继续之前，思考这些问题：

1. 为什么 Agent 需要循环，而不是单次 LLM 调用？
2. 如果上下文窗口满了，会发生什么？有什么解决方案？
3. 工具定义中的 `input_schema` 为什么要指定 `required` 字段？
4. 系统提示和消息历史有什么区别？为什么需要分开？
5. 流式处理和批量处理各适用于什么场景？

---

< 上一章 | [目录](./index.md) | 下一章：[s01-Agent 循环](./s01-the-agent-loop.md) >
