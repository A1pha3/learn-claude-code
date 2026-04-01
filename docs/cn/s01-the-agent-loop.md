# s01：Agent 循环 —— 最小可用 Agent

> **核心口号**：*"一个循环 + 一个工具 = Agent"*

> **学习目标**：理解 Agent 的本质，用不到 50 行代码实现第一个可工作的 AI 编程助手

---

## 学习目标

完成本章后，你将能够：

1. **理解 Agent 的核心机制** —— 消息循环的本质
2. **实现最小可用 Agent** —— 基于真实源码，理解每个设计决策
3. **掌握 `stop_reason` 的作用** —— 理解循环控制的关键
4. **理解消息累积模式** —— 为什么需要消息列表
5. **理解安全防护机制** —— 危险命令拦截与超时控制

**学习分层参考**：

| 层级 | 目标 | 对应内容 |
|------|------|----------|
| ⭐ 基础 | 理解 Agent 循环的四行伪代码，跑通最小 Agent | 第 1-3 节、第 5 节 |
| ⭐⭐ 进阶 | 能解释 `stop_reason`、消息累积、安全防护的每一个设计决策 | 第 4 节、第 6 节 |
| ⭐⭐⭐ 专家 | 能从零改写循环为生产级 Agent，补充重试/日志/成本追踪 | 第 7-9 节、练习 1-4 |

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
4. 输入 `sudo ls` 或 `rm -rf /` 等危险命令时观察拦截效果。

---

## 1. 问题的本质

### 1.1 LLM 的困境

想象你是一个被困在房间里绝世聪明的顾问：

```
+---------------------+
|                     |
|   LLM（你）         |
|                     |
|   "我想知道         |
|    requirements.txt |
|    的内容..."       |
|                     |
+---------------------+

X 没有手
X 没有眼睛
X 没有耳朵
X 无法接触外部世界
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
  1. 调用 bash("cat requirements.txt")
  2. 获得结果：anthropic==0.18.0
  3. 调用 bash("pip install -r requirements.txt")
  4. 获得结果：Successfully installed
  5. 返回给用户：依赖已安装完成
```

**关键**：Agent 自动处理"调用 -> 获取结果 -> 继续思考"的循环，无需用户介入。

---

## 2. 核心设计

### 2.1 架构图

源码中的数据流如下：

```
+----------+      +-------+      +---------+
|   User   | ---> |  LLM  | ---> |  Tool   |
|  prompt  |      |       |      | execute |
+----------+      +---+---+      +----+----+
                       ^               |
                       |   tool_result |
                       +---------------+
                       (loop continues)
```

更详细的流程：

```
用户输入
   |
   v
+-----------+
| 消息列表   | <-------+
| messages  |         |
+-----+-----+         |
      |               |
      v               |
+-----------+         |
|   LLM    |         | append
| API call |         |
+-----+-----+         |
      |               |
      v               |
+---------------+     |
| stop_reason?  |     |
+------+--------+     |
       |              |
   "end_turn"         |
       |              |
       v              |
   返回结果            |
       |              |
   "tool_use"         |
       |              |
       v              |
+-----------+         |
| 执行工具  |         |
| run_bash  |         |
+-----+-----+         |
      |               |
      v               |
+-----------+         |
| 收集结果  | ---------+
+-----------+   append to messages
```

### 2.2 核心模式（四行伪代码）

整个 Agent 的秘密浓缩为一个模式：

```python
while stop_reason == "tool_use":
    response = LLM(messages, tools)
    execute tools
    append results
```

这就是全部。生产级 Agent 在此基础上叠加策略（policy）、钩子（hooks）和生命周期管理（lifecycle controls），但核心循环不变。

---

## 3. 源码分析

### 3.1 环境初始化

```python
import os
import subprocess

from anthropic import Anthropic
from dotenv import load_dotenv

load_dotenv(override=True)

if os.getenv("ANTHROPIC_BASE_URL"):
    os.environ.pop("ANTHROPIC_AUTH_TOKEN", None)

client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
MODEL = os.environ["MODEL_ID"]
```

**设计要点**：

| 组件 | 作用 | 说明 |
|------|------|------|
| `load_dotenv(override=True)` | 加载 `.env` 文件 | `override=True` 确保环境变量被覆盖 |
| `ANTHROPIC_BASE_URL` | 支持自定义 API 端点 | 用于代理或兼容 API（如 OpenRouter） |
| `os.environ.pop` | 清理冲突的认证令牌 | 避免两种认证方式同时存在时的冲突 |
| `MODEL_ID` | 从环境变量读取模型名 | 灵活切换不同模型 |

**注意**：源码并未使用 `ANTHROPIC_API_KEY`，而是通过 `base_url` 配置连接，这允许对接任意兼容 API。

### 3.2 系统提示与工具定义

```python
SYSTEM = f"You are a coding agent at {os.getcwd()}. Use bash to solve tasks. Act, don't explain."

TOOLS = [{
    "name": "bash",
    "description": "Run a shell command.",
    "input_schema": {
        "type": "object",
        "properties": {"command": {"type": "string"}},
        "required": ["command"],
    },
}]
```

**系统提示分析**：

- `f"You are a coding agent at {os.getcwd()}"` -- 告诉 LLM 当前工作目录，使其生成的路径有意义
- `"Use bash to solve tasks"` -- 引导 LLM 使用工具而非空谈
- `"Act, don't explain"` -- 减少不必要的解释，直接执行

**工具定义结构**（Anthropic API 规范）：

```python
{
    "name": "bash",                    # 工具名称，LLM 调用时使用
    "description": "Run a shell command.",  # 工具描述，帮助 LLM 理解何时使用
    "input_schema": {                   # JSON Schema 格式的参数定义
        "type": "object",
        "properties": {
            "command": {"type": "string"}
        },
        "required": ["command"],
    },
}
```

### 3.3 工具执行函数 `run_bash`

```python
def run_bash(command: str) -> str:
    dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
    if any(d in command for d in dangerous):
        return "Error: Dangerous command blocked"
    try:
        r = subprocess.run(command, shell=True, cwd=os.getcwd(),
                           capture_output=True, text=True, timeout=120)
        out = (r.stdout + r.stderr).strip()
        return out[:50000] if out else "(no output)"
    except subprocess.TimeoutExpired:
        return "Error: Timeout (120s)"
```

**安全防护层**：

| 防护 | 实现 | 说明 |
|------|------|------|
| 危险命令黑名单 | `["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]` | 子字符串匹配拦截 |
| 超时控制 | `timeout=120` | 120 秒超时，防止无限等待 |
| 输出截断 | `out[:50000]` | 防止单个工具结果消耗过多上下文 |
| 空输出处理 | `"(no output)"` | 确保总有返回值，LLM 不会困惑 |

**关于 `subprocess.run` vs `os.popen`**：

源码使用 `subprocess.run` 而非 `os.popen`，这是正确的选择：

- `subprocess.run` 是 Python 3.5+ 推荐的子进程调用方式
- `capture_output=True` 同时捕获 stdout 和 stderr
- `text=True` 直接返回字符串而非字节
- `timeout` 参数原生支持超时控制
- `os.popen` 已被官方标记为过时（deprecated），且无法捕获 stderr、无法设置超时

**危险命令黑名单的实际内容**：

```
"rm -rf /"  -- 递归删除根目录
"sudo"      -- 提权操作
"shutdown"  -- 关机
"reboot"    -- 重启
"> /dev/"   -- 重定向到设备文件（可能用于数据破坏）
```

这是最基础的黑名单。生产环境中需要更严格的控制（如白名单模式、命令审计等），s02 会进一步讨论。

### 3.4 核心循环 `agent_loop`

```python
def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        # Append assistant turn
        messages.append({"role": "assistant", "content": response.content})
        # If the model didn't call a tool, we're done
        if response.stop_reason != "tool_use":
            return
        # Execute each tool call, collect results
        results = []
        for block in response.content:
            if block.type == "tool_use":
                print(f"\033[33m$ {block.input['command']}\033[0m")
                output = run_bash(block.input["command"])
                print(output[:200])
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": output})
        messages.append({"role": "user", "content": results})
```

**逐段解析**：

**第 1 步：调用 LLM**

```python
response = client.messages.create(
    model=MODEL, system=SYSTEM, messages=messages,
    tools=TOOLS, max_tokens=8000,
)
```

- 发送完整历史 `messages` 给 LLM
- `tools=TOOLS` 告诉 LLM 有哪些可用工具
- `max_tokens=8000` 限制单次响应的最大 token 数

**第 2 步：保存 LLM 响应**

```python
messages.append({"role": "assistant", "content": response.content})
```

- 将 LLM 的响应添加到消息历史
- `response.content` 是一个列表，可能包含 `text` 块和 `tool_use` 块

**第 3 步：检查循环条件**

```python
if response.stop_reason != "tool_use":
    return
```

- `stop_reason` 是循环的"方向盘"
- 只有 `"tool_use"` 时才继续循环
- 其他值（`"end_turn"`, `"max_tokens"`）都会退出

**第 4 步：执行工具并收集结果**

```python
results = []
for block in response.content:
    if block.type == "tool_use":
        print(f"\033[33m$ {block.input['command']}\033[0m")
        output = run_bash(block.input["command"])
        print(output[:200])
        results.append({"type": "tool_result", "tool_use_id": block.id,
                        "content": output})
```

- 遍历 LLM 响应中的每个块（block）
- 只处理 `tool_use` 类型的块
- 用 `\033[33m` ANSI 转义码以黄色打印命令（可视化调试）
- 截取前 200 字符打印到终端（避免刷屏）
- **完整输出**（最多 50000 字符）存入 `results`

**第 5 步：将结果添加到消息历史**

```python
messages.append({"role": "user", "content": results})
```

- 工具结果以 `"user"` 角色添加
- 这是 Anthropic API 的规范：工具结果紧跟在 assistant 的工具调用之后

### 3.5 主入口：多轮对话支持

```python
if __name__ == "__main__":
    history = []
    while True:
        try:
            query = input("\033[36ms01 >> \033[0m")
        except (EOFError, KeyboardInterrupt):
            break
        if query.strip().lower() in ("q", "exit", ""):
            break
        history.append({"role": "user", "content": query})
        agent_loop(history)
        response_content = history[-1]["content"]
        if isinstance(response_content, list):
            for block in response_content:
                if hasattr(block, "text"):
                    print(block.text)
        print()
```

**设计分析**：

| 组件 | 作用 |
|------|------|
| `history = []` | 全局消息历史，跨轮次保留 |
| `input("\033[36ms01 >> \033[0m")` | 青色提示符的交互式输入 |
| `agent_loop(history)` | 循环内部可能执行多轮工具调用 |
| `history[-1]["content"]` | 读取最后一轮 assistant 的响应并显示 |
| `hasattr(block, "text")` | 处理响应中可能包含的文本块 |

**关键洞察**：`agent_loop` 处理"工具调用"这一层的循环，而外层 `while True` 处理"用户对话"这一层的循环。两层循环共同构成了完整的交互体验。

---

## 4. 关键概念详解

### 4.1 stop_reason 的作用

`stop_reason` 是循环控制的关键：

| stop_reason | 含义 | 行动 |
|-------------|------|------|
| `tool_use` | LLM 想要调用工具 | 执行工具，继续循环 |
| `end_turn` | LLM 完成了响应 | 退出循环，返回结果 |
| `max_tokens` | 达到 Token 上限 | 需要处理（通常是截断或错误） |

**为什么叫 `tool_use` 而不是 `call_tool`？**

因为 LLM 的响应"使用"了工具，但不一定是"调用"工具。它只是表达了调用意图，真正的执行由外部代码完成。这是一种关注点分离的设计。

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

LLM 是无状态的（stateless），每次 API 调用都是独立的。要让它"记住"之前的对话，必须在每次调用时发送完整历史。这正是消息列表持续累积的原因。

### 4.3 工具结果的格式

工具结果必须匹配对应的工具调用：

```python
{
    "type": "tool_result",      # 必须是 tool_result
    "tool_use_id": block.id,    # 必须匹配原工具调用的 id
    "content": "命令的输出"      # 实际的结果内容
}
```

**`tool_use_id` 的作用**：让 LLM 知道这个结果对应哪个工具调用。当一次 LLM 响应包含多个工具调用时（并行调用），ID 匹配至关重要。

### 4.4 消息列表的数据结构

```python
messages = [
    # 轮次 1
    {"role": "user", "content": "用户的问题"},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "tool_1", "name": "bash", "input": {"command": "ls"}}
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "tool_1", "content": "file1.py\nfile2.py"}
    ]},

    # 轮次 2
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "tool_2", "name": "bash", "input": {"command": "cat file1.py"}}
    ]},
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "tool_2", "content": "print('hello')"}
    ]},
]
```

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
        tool_use: bash("ls *.py")
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
        tool_use: bash("echo 'print(\"Hello\")' > hello.py")

轮次 2：
  LLM：现在运行它
        tool_use: bash("python hello.py")

轮次 3：
  LLM：程序已运行，输出是 "Hello"
  返回
```

**关键点**：Agent 自动处理了多个步骤，用户只说了一次话。

### 5.3 示例 3：安全拦截

**用户输入**：
```
删除系统文件
```

**执行流程**：

```
轮次 1：
  LLM：需要执行删除命令
        tool_use: bash("sudo rm -rf /")
  系统：检测到 "sudo" 和 "rm -rf /" 匹配黑名单
        返回："Error: Dangerous command blocked"

轮次 2：
  LLM：命令被阻止了，我无法执行这个危险操作
  返回
```

---

## 6. 常见问题

### Q1：为什么工具结果作为"用户"角色？

```python
messages.append({"role": "user", "content": results})  # 为什么是 user？
```

**答案**：在 Anthropic 的 API 规范中，对话角色只有 `user` 和 `assistant` 两种。工具结果不是 `assistant` 说的，所以必须作为 `user` 角色提交。本质上，工具结果是"系统"反馈给 LLM 的信息，在 API 语义中以 `user` 角色承载。

### Q2：如果工具执行失败怎么办？

```python
r = subprocess.run(command, shell=True, cwd=os.getcwd(),
                   capture_output=True, text=True, timeout=120)
out = (r.stdout + r.stderr).strip()
```

**答案**：源码将 `stdout` 和 `stderr` 合并返回。如果命令不存在或执行失败，错误信息会通过 `stderr` 传递给 LLM，让 LLM 自己决定如何处理。这保持了 Agent 的自主性 -- LLM 看到错误后会尝试修复。

### Q3：为什么要限制输出长度？

```python
return out[:50000] if out else "(no output)"
```

**答案**：
1. 防止单个工具结果消耗过多上下文窗口
2. LLM 的上下文窗口有限（8000-200000 token），输出过大会导致后续轮次无法携带完整历史
3. 大多数情况下，50000 字符足够理解命令结果
4. 如果需要更多内容，LLM 可以用 `head`/`tail` 等命令读取特定部分

### Q4：循环最多执行多少次？

```python
while True:  # 无限循环？
```

**答案**：源码中没有显式的循环上限。实际上不是无限的，因为：
1. LLM 最终会停止调用工具（`stop_reason != "tool_use"`）
2. 但生产环境中应该添加最大循环次数限制

改进建议：
```python
MAX_ROUNDS = 50
def agent_loop(messages: list, max_rounds: int = MAX_ROUNDS):
    for _ in range(max_rounds):
        response = client.messages.create(...)
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return
        # ... 执行工具
    print("Warning: Reached max rounds")
```

### Q5：为什么用 `subprocess.run` 而不是 `os.popen`？

**答案**：
- `subprocess.run` 是 Python 3.5+ 推荐的子进程调用方式，`os.popen` 已被官方标记为过时
- `subprocess.run` 可以同时捕获 stdout 和 stderr（`capture_output=True`）
- 原生支持超时控制（`timeout` 参数）
- 更好的错误处理和返回码检查

---

## 7. 调试技巧

### 7.1 源码中已内置的调试

源码在工具执行时已经打印了调试信息：

```python
print(f"\033[33m$ {block.input['command']}\033[0m")  # 黄色显示命令
print(output[:200])                                    # 显示前 200 字符输出
```

### 7.2 打印消息历史

```python
def debug_print(messages):
    for i, msg in enumerate(messages):
        role = msg['role']
        content = msg['content']
        if isinstance(content, str):
            print(f"[{i}] {role}: {content[:100]}...")
        elif isinstance(content, list):
            for block in content:
                if hasattr(block, 'type'):
                    print(f"[{i}] {role}: {block.type}")
```

### 7.3 打印 stop_reason

在 `agent_loop` 中添加：
```python
print(f"stop_reason: {response.stop_reason}")
```

### 7.4 追踪消息增长

```python
print(f"Messages count: {len(messages)}, total chars: {sum(len(str(m)) for m in messages)}")
```

---

## 8. 架构分析与使用场景

### 8.1 架构特点

| 特点 | 分析 |
|------|------|
| **单工具架构** | 只有 `bash`，LLM 通过 shell 完成一切 |
| **同步阻塞** | 工具串行执行，等待结果后才继续 |
| **无状态 LLM** | 每次调用独立，通过消息列表保持上下文 |
| **有限安全** | 黑名单 + 超时，基本但关键的安全层 |
| **输出截断** | 50000 字符限制，防止上下文溢出 |

### 8.2 使用场景

| 场景 | 适用性 | 说明 |
|------|--------|------|
| 本地文件操作 | 适用 | 通过 bash 命令操作文件 |
| 代码执行与调试 | 适用 | 运行 Python/Node 脚本 |
| 系统管理 | 受限 | 黑名单拦截了 sudo/shutdown |
| 需要精确文件编辑 | 不适用 | 通过 bash echo 写文件不够可靠，s02 会改进 |
| 高安全性要求 | 不适用 | 仅黑名单防护，s02 会引入路径沙盒 |

### 8.3 与后续章节的关系

```
s01: while stop_reason == "tool_use": ...
s02: while stop_reason == "tool_use": ...  # 相同，但多了工具
s03: while stop_reason == "tool_use": ...  # 相同，但多了规划
...
s12: while stop_reason == "tool_use": ...  # 仍然相同
```

**核心洞察**：所有后续章节都是在循环"之前"和"之后"添加逻辑，而不是改变循环本身。循环结构是这个架构中唯一不变的部分。

---

## 实战应用：从最小 Agent 到你的项目

学完循环原理后，我们来看看这个不到 50 行的最小 Agent 如何映射到真实的业务场景，以及从"能跑"到"能用"还差哪些能力。

### 三大真实场景

**场景 1：自动化测试执行器**

- **背景**：团队每次提交代码后需要运行测试套件并汇总结果。
- **为什么用 Agent**：测试命令可能失败、需要重试、需要解析输出提取失败用例——这些决策交给 LLM 比硬编码脚本更灵活。
- **涉及能力**：`bash` 工具执行 `pytest` / `go test` 等命令，LLM 解析输出并决定是否重跑、如何报告。

**场景 2：日志分析助手**

- **背景**：生产环境的日志文件动辄数万行，需要快速定位错误模式并给出修复建议。
- **为什么用 Agent**：错误模式千变万化，LLM 能根据上下文推断根因并给出针对性建议，而非简单 grep。
- **涉及能力**：`bash` 工具读取日志文件（`tail` / `grep`），LLM 识别模式、定位关键行、生成修复方案。

**场景 3：文件批量处理器**

- **背景**：需要按特定规则对一批文件执行重命名、格式转换或内容整理。
- **为什么用 Agent**：规则可能因文件类型不同而变化，LLM 能根据文件内容动态选择处理策略。
- **涉及能力**：`bash` 工具执行 `mv` / `sed` / `python` 脚本，LLM 逐个判断并执行。

### 从最小 Agent 到生产 Agent 的距离

| 能力 | s01 已有 | 生产必须 | 需要补充 | 对应章节 |
|------|----------|----------|----------|----------|
| 错误重试 | 无 | 必须 | LLM 调用失败时自动重试 | -- |
| 循环上限 | 无 | 必须 | `MAX_ROUNDS` 防止无限循环 | s01 练习 2 |
| 日志记录 | 仅 `print` | 必须 | 结构化日志（JSON / 时间戳 / 轮次） | -- |
| 成本追踪 | 无 | 推荐 | Token 计数与费用估算 | -- |
| 多工具支持 | 1 个（bash） | 必须 | 文件读写、搜索等专用工具 | s02 |
| 并发执行 | 同步阻塞 | 可选 | 多工具并行调用 | s08 |

这张清单可以帮助你评估：当前 Agent 离生产部署还差多远。

### 10 分钟改造指南：替换 `run_bash` 为你自己的工具

最小 Agent 的 `run_bash` 本质上是一个"输入命令、输出字符串"的函数。你可以把它替换成任何符合这个签名的函数：

```python
# 原始：bash 命令执行器
def run_bash(command: str) -> str:
    # ...

# 替换为：HTTP 请求执行器
def run_http(request_json: str) -> str:
    import requests, json
    req = json.loads(request_json)
    resp = requests.request(req["method"], req["url"], json=req.get("body"))
    return resp.text[:50000]

# 替换为：SQL 查询执行器
def run_sql(query: str) -> str:
    import sqlite3, json
    conn = sqlite3.connect("my.db")
    rows = conn.execute(query).fetchall()
    return json.dumps(rows, ensure_ascii=False)[:50000]

# 替换为：文件系统操作器
def run_fs(operation: str) -> str:
    # operation 可以是 "list /tmp", "read /tmp/a.txt" 等
    # ...
```

**改造步骤**：

1. 保留 `agent_loop` 和消息列表机制不变。
2. 将 `TOOLS` 定义中的 `bash` 替换为你的工具 schema。
3. 将 `run_bash` 替换为你的处理函数。
4. 调整 `SYSTEM` 提示，告诉 LLM 新工具的用法和限制。
5. 运行测试，观察 LLM 如何使用新工具。

核心循环不需要改——这就是 Agent 架构的威力。

---

## 9. 小结

### 9.1 核心要点

| 要点 | 说明 |
|------|------|
| **Agent 的本质** | LLM + 工具 + 循环 |
| **循环控制** | `stop_reason == "tool_use"` 时继续 |
| **消息列表** | 完整的对话历史，每次调用都发送 |
| **工具结果** | 作为 `"user"` 角色添加回消息列表 |
| **安全防护** | 黑名单拦截危险命令 + 120 秒超时 |
| **输出控制** | 50000 字符截断，防止上下文溢出 |

### 9.2 代码模板

```python
def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": output})
        messages.append({"role": "user", "content": results})
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

```python
def agent_loop(messages: list):
    round_num = 0
    while True:
        round_num += 1
        print(f"--- Round {round_num}, messages: {len(messages)} ---")
        # ... 原有逻辑
```

### 练习 2：添加循环上限

添加 `MAX_ROUNDS` 常量，限制最大循环次数，防止无限循环。

### 练习 3：扩展危险命令黑名单

在 `run_bash` 中增加更多危险命令的拦截，例如 `mkfs`、`dd if=`、`chmod 777` 等。

### 练习 4：多工具并行

当 LLM 在一次响应中调用多个工具时（多个 `tool_use` 块），验证所有工具都被正确执行。尝试让 LLM 同时执行两个命令。

---

## 11. 下一步

你已经实现了第一个 Agent！虽然它只有一个 `bash` 工具，但核心结构已经完整。

在 [s02：工具系统](./s02-tool-use.md) 中，我们将学习：
- 如何添加多个工具（read_file、write_file、edit_file）
- 如何使用分发器模式替代 if/elif 链
- 如何实现路径沙盒防止文件系统逃逸
- 如何让工具系统保持循环结构不变

---

< 上一章：[00-基础篇](./00-basics.md) | [目录](./index.md) | 下一章：[s02-工具系统](./s02-tool-use.md) >
