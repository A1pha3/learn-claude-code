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
2. 为什么"工具调用"是 Agent 必需能力？
3. 为什么必须用循环把"思考—执行—反馈"串起来？

### 本章完成标准

- 能用自己的话解释"LLM"与"Agent"在能力边界上的区别。
- 能画出最小工具调用闭环并说明每一步职责。
- 能把本章概念映射到后续章节的具体机制。

---

## 1. 什么是 LLM（大语言模型）

### 1.1 精确定义

**大语言模型（Large Language Model, LLM）** 是一个基于 Transformer 架构的**自回归文本生成模型**。它接收一段文本作为输入（称为 Prompt），逐个 Token 预测并生成后续文本。

```
工作原理（简化）：

输入文本 → Token 化 → Transformer 计算 → 概率分布 → 采样下一个 Token → 重复

示例：
输入："法国的首都是"
预测："巴黎" (概率 0.99)
```

关键特征：
- **无状态**：每次调用都是独立的，LLM 不"记住"之前的对话
- **文本到文本**：只能接收和生成文本，不能直接操作文件系统或网络
- **概率模型**：输出基于概率分布，同一输入可能产生不同输出

### 1.2 LLM 能做什么

| 能力 | 说明 | 示例 |
|------|------|------|
| **理解** | 理解自然语言指令 | "帮我写一个冒泡排序" |
| **推理** | 进行逻辑推导 | "如果 A > B 且 B > C，那么 A 和 C 的关系是？" |
| **生成** | 创造新内容 | "写一首关于春天的诗" |
| **转换** | 改变内容形式 | "把这段代码从 Python 改成 JavaScript" |
| **总结** | 提取关键信息 | "用三句话总结这篇文章" |
| **规划** | 将大任务分解为步骤 | "帮我规划一个 Web 应用的开发步骤" |

### 1.3 LLM 的根本局限

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
   - 生成操作的指令（通过工具调用）
   - 解释操作的结果
```

**类比**：LLM 就像一个被困在房间里的绝世聪明顾问。他可以给你最好的建议，但无法帮你去倒一杯水。你需要成为他的"手"，执行他建议的操作。

**为什么这个局限很重要**：理解了这个局限，就理解了为什么需要 Agent——Agent 让 LLM 拥有了"手"。

---

## 2. 工具调用（Tool Use）

### 2.1 什么是工具调用

**工具调用（Tool Use）** 是 LLM 与外部世界交互的**标准化协议**。它不是让 LLM 直接执行操作，而是让 LLM **表达执行操作的意图**，由外部代码负责实际执行。

这个协议包含四个步骤：

```
1. 注册工具 → 告诉 LLM 有哪些工具可用
2. 表达意图 → LLM 在响应中包含工具调用请求
3. 执行操作 → 外部代码执行工具并收集结果
4. 反馈结果 → 将执行结果返回给 LLM 继续推理
```

### 2.2 工具调用的精确格式

以 Anthropic API 为例，工具调用涉及三种消息类型：

**类型 1：LLM 请求调用工具（tool_use）**

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01A09q90qw90lq917835lq9",
      "name": "read_file",
      "input": {
        "path": "requirements.txt"
      }
    }
  ]
}
```

关键字段说明：
- `type: "tool_use"` —— 标识这是一个工具调用请求
- `id` —— 唯一标识符，用于将结果与请求配对
- `name` —— 要调用的工具名称
- `input` —— 工具参数（符合工具定义的 JSON Schema）

**类型 2：系统返回工具结果（tool_result）**

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
      "content": "anthropic==0.18.0\npython-dotenv==1.0.0"
    }
  ]
}
```

关键点：`tool_use_id` 必须与请求中的 `id` 一致，这样 LLM 才知道这个结果对应哪个工具调用。

> **为什么工具结果是 `role: "user"`？** 在 Anthropic 的消息格式中，只有 `user` 和 `assistant` 两种角色。工具结果不是 LLM 生成的，所以不能是 `assistant`；它是由系统提供的补充信息，在消息格式中被归类为 `user` 角色。

**类型 3：LLM 基于结果继续思考（text）**

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "我看到项目使用 anthropic 0.18.0 版本..."
    }
  ]
}
```

### 2.3 工具定义

工具需要在调用 LLM 之前注册，告诉 LLM 有哪些工具可用、每个工具接受什么参数。工具定义使用 JSON Schema 格式：

```python
# 实际源码中的工具定义（agents/s02_tool_use.py）
TOOLS = [
    {
        "name": "bash",
        "description": "Run a shell command.",
        "input_schema": {
            "type": "object",
            "properties": {
                "command": {"type": "string"}
            },
            "required": ["command"]
        }
    },
    {
        "name": "read_file",
        "description": "Read file contents.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "limit": {"type": "integer"}
            },
            "required": ["path"]
        }
    },
    {
        "name": "write_file",
        "description": "Write content to file.",
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
        "name": "edit_file",
        "description": "Replace exact text in file.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "old_text": {"type": "string"},
                "new_text": {"type": "string"}
            },
            "required": ["path", "old_text", "new_text"]
        }
    }
]
```

**设计要点**：
1. `name`：LLM 调用时使用的标识符，对应分发器中的键
2. `description`：帮助 LLM 理解何时使用这个工具
3. `input_schema`：使用 JSON Schema 定义参数类型和约束
4. `required`：标记哪些参数是必需的

**关键点**：工具定义是 LLM **唯一能看到的"能力清单"**。它只能调用已定义的工具，无法调用未注册的工具。

### 2.4 stop_reason：循环控制的关键

每次 LLM 响应都会返回一个 `stop_reason`，告诉我们为什么停止生成。这是 Agent 循环控制的核心机制：

| stop_reason | 含义 | 下一步操作 |
|-------------|------|------------|
| `tool_use` | LLM 想要调用工具 | 执行工具，返回结果，继续循环 |
| `end_turn` | LLM 完成了响应 | 将结果返回给用户，退出循环 |
| `max_tokens` | 达到 Token 上限 | 需要处理（压缩上下文或增加 max_tokens） |

**在源码中的使用**（agents/s01_agent_loop.py 第 76 行）：

```python
if response.stop_reason != "tool_use":
    return  # 退出循环
```

**核心机制**：`stop_reason == "tool_use"` 是 Agent 循环继续的**唯一条件**。只要 LLM 还在请求调用工具，循环就继续；一旦 LLM 认为任务完成（`end_turn`），循环就停止。

---

## 3. 上下文窗口

### 3.1 什么是上下文窗口

**上下文窗口（Context Window）** 是 LLM 一次能处理的**最大 Token 数量**，包括输入和输出。它决定了 Agent 的"短期记忆"容量。

| 模型系列 | 上下文窗口 |
|----------|------------|
| Claude 3 / 3.5 系列 | 200,000 tokens |
| Claude 4 系列 | 200,000 tokens |

### 3.2 Token 的概念

**Token** 是 LLM 处理文本的基本单位。Token 不等于单词或字符：

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

每次调用 LLM，需要发送完整的消息历史。上下文窗口中的内容构成：

```
上下文窗口 = 系统提示 + 消息历史 + 本次输出

其中消息历史：
messages = [
    {"role": "user", "content": "用户的问题"},           # 轮次 1 开始
    {"role": "assistant", "content": [ToolUse(...)]},    # LLM 请求工具
    {"role": "user", "content": [ToolResult(...)]},      # 工具结果
    {"role": "assistant", "content": [ToolUse(...)]},    # LLM 再次请求
    {"role": "user", "content": [ToolResult(...)]},      # 再次结果
    # ... 随对话增长
]
```

**问题**：随着对话进行，消息历史不断增长，最终可能超出上下文窗口限制。

**各阶段的大致消耗**：

```
系统提示：~200 tokens（固定成本）
用户问题：~100 tokens
每次工具调用：~50 tokens（工具名 + 参数）
每次工具结果：~500-5000 tokens（取决于输出大小）
一轮完整交互：~1000-10000 tokens

一个典型的编程会话（20 轮）：~50,000-100,000 tokens
```

### 3.4 解决方案预览

本课程提供了多层解决方案：

| 方案 | 章节 | 触发时机 | 核心思想 |
|------|------|----------|----------|
| 上下文隔离 | s04 | 委派子任务时 | 给子任务独立的上下文，不污染主上下文 |
| 按需加载 | s05 | 需要知识时 | 不预先加载所有知识，只在需要时加载 |
| 三层压缩 | s06 | 上下文接近饱和时 | 清理旧内容，保留关键信息 |

---

## 4. 流式 vs 批量处理

### 4.1 批量处理（Blocking）

等待完整响应后再处理：

```python
response = client.messages.create(
    model=MODEL,
    messages=messages,
    max_tokens=8000,
)
# 等待...可能几秒钟
print(response.content)  # 一次性获得完整响应
```

**优点**：简单，能获取完整的 `stop_reason` 来决定下一步
**缺点**：用户等待时间长，体验差

### 4.2 流式处理（Streaming）

逐块接收响应：

```python
# 注意：MODEL 变量在前面的初始化模式中已定义（MODEL = os.environ["MODEL_ID"]）
with client.messages.stream(
    model=MODEL,
    messages=messages,
    max_tokens=8000,
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)  # 实时输出
```

**优点**：用户立即看到输出，体验好
**缺点**：需要积累完整响应才能判断 `stop_reason`

### 4.3 Agent 中的选择

**对于 Agent 构建，本课程使用批量处理**，原因有三：

1. **需要完整的 stop_reason**：Agent 循环的核心是 `if response.stop_reason != "tool_use": return`，必须等响应完成才能判断
2. **需要完整的工具调用**：工具调用的参数可能跨越多个流式块，完整响应更容易处理
3. **简化错误处理**：批量模式下重试和错误恢复更直接

> **但在面向用户的聊天界面中**，流式处理是必须的——用户需要实时看到 Agent 的思考过程。生产系统通常使用流式处理接收 LLM 输出，但在循环控制层面仍需等待完整的 `stop_reason`。

---

## 5. Agent 的核心模式

### 5.1 精确定义

**Agent = LLM + 工具 + 循环**

更精确地说，Agent 是一个自主循环系统，以 LLM 为决策核心，通过工具与外部世界交互，循环直到任务完成。

三个要素缺一不可：
- **LLM**：理解意图、制定计划、决策下一步操作
- **工具**：将 LLM 的决策转化为实际操作
- **循环**：自动处理"调用 → 获取结果 → 继续思考"的闭环

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   ┌───────┐      ┌───────┐      ┌───────────┐     │
│   │ 用户   │ ---> │  LLM  │ ---> │   工具    │     │
│   │ 提问   │      │ 决策  │ <--- │   执行    │     │
│   └───────┘      └───────┘      └───────────┘     │
│                          ↑                          │
│                          │                          │
│                    工具结果                        │
│                          │                          │
│                    (循环继续)                       │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 5.2 消息循环——Agent 的核心代码

以下是 Agent 的核心伪代码，对应源码 `agents/s01_agent_loop.py` 的 `agent_loop()` 函数：

```python
def agent_loop(messages: list):
    while True:
        # 1. 调用 LLM
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        # 2. 保存 LLM 响应到消息历史
        messages.append({"role": "assistant", "content": response.content})

        # 3. 检查是否继续循环
        if response.stop_reason != "tool_use":
            return  # LLM 完成工作，退出循环

        # 4. 执行工具并收集结果
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output
                })

        # 5. 将结果添加到消息历史，继续循环
        messages.append({"role": "user", "content": results})
```

**这个模板贯穿整个课程，从 s01 到 s12 保持不变。**

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

**核心洞察**：循环让 LLM 从"被动回答问题的顾问"变成"主动完成任务的助手"。

### 5.4 本课程的循环不变原则

> Agent 循环一旦建立，后续所有机制都不修改循环本身。

```
s01: while stop_reason == "tool_use": ...   # 建立循环
s02: while stop_reason == "tool_use": ...   # 相同，只是 TOOLS 更多
s03: while stop_reason == "tool_use": ...   # 相同，增加了提醒计数器
s04: while stop_reason == "tool_use": ...   # 相同，增加了子 Agent 调度
...
s12: while stop_reason == "tool_use": ...   # 仍然相同
```

所有后续章节都是在循环**之前**（如压缩检查）和**之后**（如工具执行）添加逻辑，而不是改变循环本身。

---

## 6. 系统提示（System Prompt）

### 6.1 什么是系统提示

**系统提示（System Prompt）** 是给 LLM 的"角色指令"，定义它的身份、行为和规则。系统提示通过 `system` 参数传入，与消息历史分离。

**实际源码示例**（agents/s01_agent_loop.py 第 40 行）：

```python
SYSTEM = f"You are a coding agent at {os.getcwd()}. Use bash to solve tasks. Act, don't explain."
```

更复杂的系统提示（agents/s03_todo_write.py 第 46-48 行）：

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Use the todo tool to plan multi-step tasks. Mark in_progress before starting, completed when done.
Prefer tools over prose."""
```

### 6.2 系统提示 vs 消息历史

| 方面 | 系统提示 | 消息历史 |
|------|----------|----------|
| 作用 | 定义角色、规则、能力 | 记录对话内容 |
| 变化 | 通常固定（或按需更新） | 每轮增长 |
| Token 占用 | 较少（几百 tokens） | 较大（随对话增长，可达数万） |
| 位置 | `system` 参数 | `messages` 参数 |
| 可见性 | 每次 LLM 调用都可见 | 可能被压缩截断 |

**设计原则**：
1. 系统提示只包含**必须的、不随对话变化的**信息
2. 动态信息（如技能详情）应该按需加载（s05 会讲解）
3. 系统提示太长会导致工具结果的相对重要性下降

---

## 7. 本课程的技术栈

本课程使用以下技术栈：

| 组件 | 技术 | 说明 |
|------|------|------|
| LLM SDK | Anthropic Python SDK (`anthropic`) | 调用 Claude API |
| 运行环境 | Python 3 + `uv` | 统一的包管理 |
| 环境变量 | `python-dotenv` | 从 `.env` 文件加载 API 密钥 |
| 文件操作 | `pathlib.Path` | 路径处理和沙盒检查 |
| 命令执行 | `subprocess.run` | 安全的 shell 命令执行 |
| 并发 | `threading` | 后台任务（s08） |
| 版本控制 | Git Worktree | 执行隔离（s12） |

### 源码中的初始化模式

每个章节的脚本都遵循相同的初始化模式：

```python
import os
from anthropic import Anthropic
from dotenv import load_dotenv

load_dotenv(override=True)

# 支持自定义 API 端点（兼容不同 LLM 提供商）
if os.getenv("ANTHROPIC_BASE_URL"):
    os.environ.pop("ANTHROPIC_AUTH_TOKEN", None)

client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
MODEL = os.environ["MODEL_ID"]
```

> **注意**：运行前需要配置 `.env` 文件，参考 `.env.example` 了解需要哪些环境变量。

---

## 8. 小结与下一步

### 8.1 核心概念回顾

| 概念 | 关键要点 |
|------|----------|
| **LLM** | 自回归文本生成模型，只能处理文本，不能直接操作外部世界 |
| **工具调用** | LLM 与外部世界交互的标准化协议（tool_use → tool_result → text） |
| **stop_reason** | `tool_use` 表示继续循环，`end_turn` 表示可以返回 |
| **上下文窗口** | 一次能处理的最大 Token 数量，需要主动管理 |
| **消息循环** | LLM → 工具 → 结果 → LLM 的 while 循环，是 Agent 的核心 |
| **系统提示** | 定义 Agent 的角色和行为规则，与消息历史分离 |

### 8.2 关键洞察

```
整个 Agent 的核心就是一个 while 循环：

while stop_reason == "tool_use":
    call_llm()        # 调用 LLM 决策
    execute_tools()   # 执行工具操作
    append_results()  # 收集结果，继续循环

所有其他机制（规划、技能、压缩、团队）都是
在这个循环的"前"和"后"添加逻辑，而不是改变循环本身。
```

### 8.3 下一步

你已经掌握了构建 Agent 所需的基础知识。在下一章 [s01-Agent 循环](./s01-the-agent-loop.md) 中，我们将用不到 30 行 Python 代码实现第一个可工作的 Agent。

---

## 新手常见问题

在实际动手运行本课程代码之前，以下是新手最容易踩的几个坑：

### 问题 1：API 密钥未配置

**症状**：运行脚本时报错 `AuthenticationError` 或 `ANTHROPIC_API_KEY not found`。

**原因**：未在项目根目录创建 `.env` 文件，或文件中缺少 `ANTHROPIC_API_KEY` 字段。

**解决**：参考项目根目录的 `.env.example` 文件，创建 `.env` 文件并填入你的 API 密钥：

```bash
cp .env.example .env
# 编辑 .env，填入你的实际密钥
```

### 问题 2：模型名填写错误

**症状**：运行时报错 `Model not found` 或 `Invalid model ID`。

**原因**：`.env` 中的 `MODEL_ID` 值与实际可用的模型名不一致（例如拼写错误、使用了不存在的前缀）。

**解决**：确认 `MODEL_ID` 使用正确的模型标识符（如 `claude-sonnet-4-20250514`），注意大小写和连字符。

### 问题 3：缺少 Python 依赖

**症状**：`ModuleNotFoundError: No module named 'anthropic'` 或类似的导入错误。

**原因**：没有使用 `uv` 来运行脚本，或虚拟环境未正确设置。

**解决**：始终使用 `uv run python agents/xxx.py` 来运行脚本，`uv` 会自动管理依赖。

### 问题 4：修改代码后未生效

**症状**：修改了 SKILL.md 或系统提示，但运行结果没有变化。

**原因**：部分内容在程序启动时一次性加载到内存中，修改文件后需要重新运行脚本才会生效。

**解决**：每次修改文件后，重新执行 `uv run python agents/xxx.py`。

---

## 思考题

在继续之前，思考这些问题：

1. 为什么 Agent 需要循环，而不是单次 LLM 调用？

   <details><summary>提示方向</summary>LLM 是无状态的——它每次调用都独立，无法自动"看到"上一次工具执行的结果。循环的作用就是让 LLM 能在拿到工具结果后继续推理，形成闭环。</details>

2. 如果上下文窗口满了，会发生什么？有什么解决方案？

   <details><summary>提示方向</summary>上下文溢出会导致早期的消息被截断或 API 报错。本课程的 s04（上下文隔离）和 s06（三层压缩）分别从"预防"和"治理"两个角度解决这个问题。</details>

3. 工具定义中的 `required` 字段有什么作用？如果漏写会怎样？

   <details><summary>提示方向</summary>required 告诉 LLM 哪些参数是必须提供的。如果漏写，LLM 可能只传可选参数而跳过关键参数，导致工具执行失败或产生意外结果。</details>

4. 系统提示和消息历史有什么区别？为什么需要分开？

   <details><summary>提示方向</summary>系统提示是每次调用都完整可见的"角色设定"，而消息历史会随对话增长，在上下文压缩时可能被截断。分开管理可以让核心指令始终可见。</details>

5. 流式处理和批量处理各适用于什么场景？

   <details><summary>提示方向</summary>面向用户的聊天界面需要流式处理（实时反馈），而 Agent 内部的循环控制更适合批量处理（需要完整的 stop_reason 来决策下一步）。</details>

---

< 上一章 | [目录](./index.md) | 下一章：[s01-Agent 循环](./s01-the-agent-loop.md) >
