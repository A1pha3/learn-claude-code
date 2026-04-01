# s02：工具系统 —— 添加工具的正确方式

> **核心口号**：*"添加工具 = 添加处理器"*

> **学习目标**：掌握多工具系统的设计模式，理解分发器的作用，实现安全的文件操作

---

## 学习目标

完成本章后，你将能够：

1. **理解分发器模式** -- 为什么不用 if/elif 链
2. **理解 Lambda + `**kw` 展开** -- 分发器如何桥接工具输入与处理函数
3. **实现安全的文件操作** -- 路径沙盒 `is_relative_to` 的原理
4. **掌握工具注册机制** -- 添加工具不影响循环结构
5. **理解工具安全设计的生产级考量** -- 从黑名单到沙盒的演进

**学习分层参考**：

| 层级 | 目标 | 对应内容 |
|------|------|----------|
| ⭐ 基础 | 理解分发器模式，能按三步流程添加新工具 | 第 1-3 节、第 7 节 |
| ⭐⭐ 进阶 | 能解释 `safe_path` 的每一行原理，设计安全的工具处理器 | 第 4 节、第 6 节 |
| ⭐⭐⭐ 专家 | 能为真实项目设计完整工具系统，平衡安全性与灵活性 | 第 5-6 节、练习 1-5 |

---

## 0. 上手演练（建议先做）

先运行工具系统示例，再回看分发器设计：

```bash
uv run python agents/s02_tool_use.py
```

建议观察：

1. 新增工具时，循环主体是否保持不变。
2. 工具定义（schema）与处理器（handler）如何一一对应。
3. 路径沙盒与参数校验在安全性上的作用。
4. 尝试让 Agent 读取 `../` 或 `/etc/passwd` 等逃逸路径，观察拦截效果。

---

## 1. 从一个工具到多个工具

### 1.1 回顾 s01

在 s01 中，我们只有一个 `bash` 工具，工具调用是硬编码的：

```python
# s01 的代码（简化）
for block in response.content:
    if block.type == "tool_use":
        command = block.input["command"]  # 硬编码：只处理 bash 的 command 参数
        output = run_bash(command)
```

**问题**：如果要添加 `read_file`、`write_file` 等工具，代码会变成什么？

### 1.2 错误的做法：if/elif 链

```python
# 不推荐：if/elif 链
for block in response.content:
    if block.type == "tool_use":
        if block.name == "bash":
            command = block.input["command"]
            output = run_bash(command)
        elif block.name == "read_file":
            path = block.input["path"]
            output = run_read(path)
        elif block.name == "write_file":
            path = block.input["path"]
            content = block.input["content"]
            output = run_write(path, content)
        elif block.name == "edit_file":
            # ... 更多代码
        # 每添加一个工具，都要修改这里！
```

**问题**：
1. 代码臃肿，难以维护
2. 添加工具需要修改核心循环逻辑
3. 违反开闭原则（对扩展开放，对修改封闭）
4. 参数提取代码重复且容易出错

### 1.3 正确的做法：分发器（Dispatcher）

s02 源码中的实际实现：

```python
# 实际源码中的分发器
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}

# 循环中的调用
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
```

**优点**：
1. 添加工具只需在字典中添加一项
2. 核心循环逻辑完全不变
3. 工具处理函数可以独立开发和测试
4. 统一的调用接口：`handler(**block.input)`

---

## 2. 分发器模式详解

### 2.1 什么是分发器

分发器是一个**工具名称到处理函数的映射字典**。它的核心职责是将 LLM 输出的工具调用请求路由到正确的处理函数。

```
LLM 响应：tool_use("read_file", {"path": "test.txt", "limit": 100})
                    |
                    v
           分发器查找：TOOL_HANDLERS["read_file"]
                    |
                    v
           找到 lambda：lambda **kw: run_read(kw["path"], kw.get("limit"))
                    |
                    v
           展开参数：run_read(path="test.txt", limit=100)
                    |
                    v
           返回结果：文件内容
```

### 2.2 Lambda + `**kw` 展开机制

源码中的分发器使用了 `lambda **kw:` 模式。这是一个精巧的设计，让我们逐步分析。

**`**kw` 的含义**：

`**kw` 在 lambda 参数中接收一个关键字参数字典。当调用 `handler(**block.input)` 时：

```python
# block.input 是 LLM 返回的参数字典
block.input = {"path": "test.txt", "limit": 100}

# handler(**block.input) 等价于：
handler(path="test.txt", limit=100)

# lambda **kw: ... 接收后，kw 变为：
kw = {"path": "test.txt", "limit": 100}

# 然后在 lambda 内部手动提取：
run_read(kw["path"], kw.get("limit"))
```

**为什么要用 lambda 而不是直接引用函数？**

对比两种方式：

```python
# 方式 A：直接引用
TOOL_HANDLERS = {
    "read_file": run_read,  # 直接引用函数
}
# 调用时：handler(**block.input)
# 等价于：run_read(path="test.txt", limit=100)
```

```python
# 方式 B：lambda 包装（源码实际使用的方式）
TOOL_HANDLERS = {
    "read_file": lambda **kw: run_read(kw["path"], kw.get("limit")),
}
# 调用时：handler(**block.input)
# lambda 接收后手动提取参数，可以控制默认值和参数映射
```

**类比：lambda 是海关检查站**

把 `block.input` 想象成一个入境旅客的行李箱。LLM 是发件方，工具函数是收件方。如果你直接引用函数（`TOOL_HANDLERS = {"read_file": run_read}`），就像让行李从航班直达收件人，中间没有任何检查 -- 行李箱里如果塞了违禁品（多余参数），收件人就会报警（`TypeError: unexpected keyword argument`）。

Lambda 就是海关：它打开行李箱（`**kw`），只挑出收件人需要的东西（`kw["path"]`, `kw.get("limit")`），其余的一律丢弃。收件人永远只收到自己申报的物品。

**源码选择 lambda 的四个原因**：

1. **参数默认值控制**：`kw.get("limit")` 而非 `kw["limit"]`，当 LLM 不传 `limit` 时返回 `None`，而 `run_read(path: str, limit: int = None)` 正好接收 `None` 作为默认值
2. **参数名映射**：如果工具 schema 的参数名与函数参数名不同，lambda 可以做映射（如 `kw["file_path"]` 映射到函数的 `path` 参数）
3. **防御性编程**：如果 `block.input` 包含函数不认识的额外参数，lambda 通过显式提取可以忽略它们，不会触发 `TypeError`
4. **接口隔离**：每个 lambda 明确声明了该工具需要哪些参数，起到了自文档化作用

**不使用 lambda 的风险**：

```python
# 如果直接引用 run_read：
TOOL_HANDLERS = {"read_file": run_read}
handler(**block.input)

# 当 block.input 包含函数不认识的参数时：
block.input = {"path": "test.txt", "limit": 100, "encoding": "utf-8"}
# run_read(**block.input) 会报 TypeError: unexpected keyword argument 'encoding'
```

Lambda 包装通过显式参数提取避免了这个问题。

### 2.3 分发器在循环中的使用

实际源码中的循环调用：

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
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                print(f"> {block.name}: {output[:200]}")
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": output})
        messages.append({"role": "user", "content": results})
```

**与 s01 的对比**：

| 方面 | s01 | s02 |
|------|-----|-----|
| 工具执行 | `output = run_bash(block.input["command"])` | `output = handler(**block.input)` |
| 工具查找 | 无（硬编码 bash） | `TOOL_HANDLERS.get(block.name)` |
| 未知工具 | 不可能（只有一种） | 返回 `"Unknown tool: ..."` |
| 循环结构 | 相同 | **完全相同** |

**核心洞察**：循环代码唯一的变化是把硬编码的 `run_bash` 替换为通用的 `handler(**block.input)`。循环的骨架没有改变。

---

## 3. 工具定义与 schema

### 3.1 源码中的完整工具定义

```python
TOOLS = [
    {"name": "bash", "description": "Run a shell command.",
     "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}},
    {"name": "read_file", "description": "Read file contents.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "limit": {"type": "integer"}}, "required": ["path"]}},
    {"name": "write_file", "description": "Write content to file.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "content": {"type": "string"}}, "required": ["path", "content"]}},
    {"name": "edit_file", "description": "Replace exact text in file.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "old_text": {"type": "string"}, "new_text": {"type": "string"}}, "required": ["path", "old_text", "new_text"]}},
]
```

### 3.2 Schema 结构分析

每个工具定义遵循 Anthropic API 的规范：

```python
{
    "name": "read_file",           # 工具名称（唯一标识符）
    "description": "...",          # 描述（帮助 LLM 理解何时使用）
    "input_schema": {              # JSON Schema 格式
        "type": "object",
        "properties": {            # 参数定义
            "path": {"type": "string"},   # 必需参数
            "limit": {"type": "integer"}  # 可选参数
        },
        "required": ["path"]       # 必需参数列表
    }
}
```

**四工具的职责分工**：

| 工具 | 参数 | 职责 |
|------|------|------|
| `bash` | `command` | 通用命令执行（带黑名单保护） |
| `read_file` | `path`, `limit?` | 安全读取文件内容 |
| `write_file` | `path`, `content` | 安全写入文件 |
| `edit_file` | `path`, `old_text`, `new_text` | 精确文本替换 |

### 3.3 工具定义与处理器的对应关系

```python
# 工具定义中的 name 必须与 TOOL_HANDLERS 的 key 一致
TOOLS = [
    {"name": "bash", ...},        # <-- 对应 --> TOOL_HANDLERS["bash"]
    {"name": "read_file", ...},   # <-- 对应 --> TOOL_HANDLERS["read_file"]
    {"name": "write_file", ...},  # <-- 对应 --> TOOL_HANDLERS["write_file"]
    {"name": "edit_file", ...},   # <-- 对应 --> TOOL_HANDLERS["edit_file"]
]

# schema 中的 properties 必须与 lambda 中的参数提取一致
{"properties": {"path": ..., "limit": ...}}
# 对应 lambda **kw: run_read(kw["path"], kw.get("limit"))
```

**如果定义与实现不一致**：
- `name` 不匹配：分发器返回 `"Unknown tool"`
- 参数不匹配：lambda 提取时 `KeyError` 或函数 `TypeError`
- 这就是为什么 lambda 模式比直接引用更安全 -- 它提供了显式的参数映射层

---

## 4. 实现安全的文件操作

### 4.1 为什么不能只靠 bash

s01 的 `bash` 工具可以执行任何命令，包括危险操作。即使有黑名单，通过 bash 操作文件仍然不够安全：

```python
# 这些命令不会被黑名单拦截，但很危险
bash("cat /etc/passwd")              # 读取系统敏感文件
bash("cd ../../; rm important.txt")  # 跳出工作目录
bash("echo '' > config.yaml")        # 清空配置文件
```

**解决方案**：使用专门的文件操作工具，在工具层面实现安全控制。

### 4.2 路径沙盒 `safe_path`

源码中的核心安全机制：

```python
from pathlib import Path

WORKDIR = Path.cwd()

def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

**逐行分析**：

```python
WORKDIR = Path.cwd()                        # 1. 记录工作目录为绝对路径
path = (WORKDIR / p).resolve()              # 2. 拼接路径并解析（消除 .. 和符号链接）
if not path.is_relative_to(WORKDIR):        # 3. 检查解析后的路径是否仍在工作目录下
    raise ValueError(f"Path escapes workspace: {p}")  # 4. 不在则抛出异常
return path                                  # 5. 返回安全的绝对路径
```

**`Path.resolve()` 的作用**：

```python
# resolve() 会解析所有 .. 和符号链接
WORKDIR = Path("/workspace")
(WORKDIR / "../etc/passwd").resolve()
# 结果：Path("/etc/passwd")  -- 不是 /workspace/../etc/passwd

(WORKDIR / "./subdir/../file.txt").resolve()
# 结果：Path("/workspace/file.txt")
```

**`is_relative_to` 的工作原理**：

`is_relative_to` 是 Python 3.9+ 引入的方法，用于检查一个路径是否是另一个路径的子路径：

```python
Path("/workspace/file.txt").is_relative_to(Path("/workspace"))    # True
Path("/workspace/sub/file.txt").is_relative_to(Path("/workspace"))  # True
Path("/etc/passwd").is_relative_to(Path("/workspace"))            # False
Path("/workspace").is_relative_to(Path("/workspace"))             # True（自身也是相对的）
```

它的实现逻辑等价于：

```python
# is_relative_to 的内部逻辑（简化）
def is_relative_to(self, other):
    try:
        self.relative_to(other)  # 如果能计算相对路径，说明 self 在 other 下
        return True
    except ValueError:
        return False
```

**关键：为什么必须在 `resolve()` 之后检查**：

```python
# 错误做法：不 resolve
path = WORKDIR / p  # 可能是 /workspace/../etc/passwd
path.is_relative_to(WORKDIR)  # True！因为字面上以 /workspace 开头

# 正确做法：先 resolve 再检查
path = (WORKDIR / p).resolve()  # /etc/passwd
path.is_relative_to(WORKDIR)    # False！正确拦截
```

**安全示例**：

```python
safe_path("test.txt")            # -> Path("/workspace/test.txt")
safe_path("./subdir/file.txt")   # -> Path("/workspace/subdir/file.txt")
safe_path("subdir/../file.txt")  # -> Path("/workspace/file.txt")  （.. 被消除）

safe_path("../escape.txt")       # -> ValueError: Path escapes workspace
safe_path("/etc/passwd")         # -> ValueError: Path escapes workspace
safe_path("subdir/../../escape") # -> ValueError: Path escapes workspace
```

### 4.3 读取文件 `run_read`

源码实现：

```python
def run_read(path: str, limit: int = None) -> str:
    try:
        text = safe_path(path).read_text()
        lines = text.splitlines()
        if limit and limit < len(lines):
            lines = lines[:limit] + [f"... ({len(lines) - limit} more lines)"]
        return "\n".join(lines)[:50000]
    except Exception as e:
        return f"Error: {e}"
```

**设计要点**：

| 要点 | 实现 | 说明 |
|------|------|------|
| 路径安全 | `safe_path(path)` | 所有路径都经过沙盒检查 |
| 行数限制 | `limit` 参数 | 防止读取过大文件 |
| 截断提示 | `"... (N more lines)"` | 告知 LLM 内容被截断 |
| 输出截断 | `[:50000]` | 防止消耗过多上下文 |
| 统一错误处理 | `try/except` | 所有异常都转为错误文本返回给 LLM |

**注意**：源码中 `run_read` 不检查文件是否存在，而是依赖 `safe_path(path).read_text()` 的自然异常（`FileNotFoundError`），由外层 `try/except` 捕获并返回错误信息。

### 4.4 写入文件 `run_write`

源码实现：

```python
def run_write(path: str, content: str) -> str:
    try:
        fp = safe_path(path)
        fp.parent.mkdir(parents=True, exist_ok=True)
        fp.write_text(content)
        return f"Wrote {len(content)} bytes to {path}"
    except Exception as e:
        return f"Error: {e}"
```

**设计要点**：

- `fp.parent.mkdir(parents=True, exist_ok=True)` -- 自动创建父目录，避免因目录不存在而写入失败
- 返回写入的字节数，帮助 LLM 确认操作成功
- 同样通过 `safe_path` 确保不会写到工作目录之外

### 4.5 编辑文件 `run_edit`

源码实现：

```python
def run_edit(path: str, old_text: str, new_text: str) -> str:
    try:
        fp = safe_path(path)
        content = fp.read_text()
        if old_text not in content:
            return f"Error: Text not found in {path}"
        fp.write_text(content.replace(old_text, new_text, 1))
        return f"Edited {path}"
    except Exception as e:
        return f"Error: {e}"
```

**设计要点**：

- `content.replace(old_text, new_text, 1)` -- 只替换第一个匹配，避免误改
- 先检查 `old_text not in content`，找不到时返回明确错误
- **为什么不用 `sed`**：Python 字符串操作比 `sed` 更可靠，不受特殊字符（正则元字符、换行符等）影响

### 4.6 bash 工具的安全防护

s02 中的 `run_bash` 保留了 s01 的黑名单机制：

```python
def run_bash(command: str) -> str:
    dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
    if any(d in command for d in dangerous):
        return "Error: Dangerous command blocked"
    try:
        r = subprocess.run(command, shell=True, cwd=WORKDIR,
                           capture_output=True, text=True, timeout=120)
        out = (r.stdout + r.stderr).strip()
        return out[:50000] if out else "(no output)"
    except subprocess.TimeoutExpired:
        return "Error: Timeout (120s)"
```

**注意**：`cwd` 参数从 s01 的 `os.getcwd()` 改为 `WORKDIR`（`Path.cwd()`），使用 `pathlib.Path` 类型保持一致。

---

## 5. 源码完整架构分析

### 5.1 模块结构

```
s02_tool_use.py
+-- 环境初始化
|   +-- load_dotenv / Anthropic client
|   +-- WORKDIR = Path.cwd()
|
+-- 安全层
|   +-- safe_path() -- 路径沙盒
|
+-- 工具实现层
|   +-- run_bash()
|   +-- run_read()
|   +-- run_write()
|   +-- run_edit()
|
+-- 注册层
|   +-- TOOL_HANDLERS -- 分发器字典
|   +-- TOOLS -- 工具 schema 列表
|
+-- 核心循环
|   +-- agent_loop()
|
+-- 主入口
    +-- if __name__ == "__main__"
```

### 5.2 数据流

```
用户输入 "读取 config.yaml"
    |
    v
agent_loop(messages)
    |
    v
LLM API 返回：
    content = [
        TextBlock("I'll read the file"),
        ToolUseBlock(id="tool_1", name="read_file", input={"path": "config.yaml"})
    ]
    stop_reason = "tool_use"
    |
    v
分发器查找：TOOL_HANDLERS["read_file"]
    |
    v
Lambda 执行：lambda **kw: run_read(kw["path"], kw.get("limit"))
    输入：{"path": "config.yaml"}
    展开为：run_read(path="config.yaml", limit=None)
    |
    v
run_read 内部：
    1. safe_path("config.yaml") -> Path("/workspace/config.yaml")
    2. read_text() -> 文件内容
    3. 截断处理 -> 返回内容
    |
    v
构建 tool_result：
    {"type": "tool_result", "tool_use_id": "tool_1", "content": "文件内容..."}
    |
    v
追加到 messages，回到 LLM
    |
    v
LLM 返回最终回答（stop_reason = "end_turn"）
```

### 5.3 与 s01 的完整对比

| 方面 | s01 | s02 |
|------|-----|-----|
| 工具数量 | 1（bash） | 4（bash, read, write, edit） |
| 工具查找 | 硬编码 `block.input["command"]` | 分发器 `TOOL_HANDLERS.get(block.name)` |
| 路径安全 | 无 | `safe_path()` 沙盒 |
| 文件操作 | 通过 bash echo/cat | 专用工具（read/write/edit） |
| 参数传递 | 直接取 `["command"]` | `handler(**block.input)` |
| 工作目录 | `os.getcwd()` 字符串 | `Path.cwd()` Path 对象 |
| 错误处理 | 无（依赖 bash 自身） | 每个工具函数内部 try/except |
| 循环结构 | `while stop_reason == "tool_use"` | **完全相同** |

### 5.4 设计决策记录

源码中每一个设计选择都不是随意的。下表汇总了关键决策及其替代方案，便于复盘和讨论：

| 决策 | 源码中的选择 | 被拒绝的替代方案 | 选择理由 |
|------|-------------|-----------------|---------|
| 工具路由 | 字典分发器 `TOOL_HANDLERS` | `if/elif` 链 | 遵循开闭原则：添加工具不修改循环代码，只追加字典条目 |
| 参数适配 | `lambda **kw:` 显式提取 | 直接引用函数 `run_read` | 隔离 LLM 输出与函数签名，防止多余参数引发 `TypeError` |
| 路径安全 | `resolve()` + `is_relative_to()` | 字符串前缀检查 `startswith()` | 字符串检查无法防御 `../` 穿越和符号链接；`resolve()` 先消除混淆再判定 |
| 错误处理 | 每个函数内部 `try/except` 返回字符串 | 异常上抛到循环层 | 保证循环永不中断；错误信息作为 `tool_result` 返回 LLM 使其可自行修正 |
| 输出截断 | `[:50000]` 硬编码 | 不截断 / 动态计算 | 防止单次工具调用耗尽上下文窗口；硬编码简单且实际够用 |
| 文本替换 | `str.replace(old, new, 1)` | `sed` 命令 | Python 字符串操作不受正则元字符干扰，行为可预测 |

> **延伸思考**：如果你要在生产系统中使用这些模式，哪些"被拒绝的替代方案"可能反而更合适？例如，当工具数量超过 50 个时，用装饰器自动注册是否比手写字典更可维护？

---

## 6. 工具安全设计的生产级考量

### 6.1 安全防护的三个层次与升级路径

安全不是非此即彼的选择，而是一条逐层升级的路径。每一层都是前一层的补充，而非替代：

```
┌──────────────────────────────────────────────────────────────────┐
│  第 3 层：容器隔离（生产级，最高安全性）                             │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  第 2 层：路径沙盒 + 专用工具（s02 实现）                       │ │
│  │  ┌──────────────────────────────────────────────────────────┐ │ │
│  │  │  第 1 层：黑名单拦截（s01 实现）                           │ │ │
│  │  │  • 拦截已知危险命令字符串                                   │ │ │
│  │  │  • 成本：极低  |  防御范围：窄                              │ │ │
│  │  └──────────────────────────────────────────────────────────┘ │ │
│  │  • 限制文件操作在 WORKDIR 内                                   │ │
│  │  • 用专用工具替代 bash 文件操作                                 │ │
│  │  • 成本：低  |  防御范围：中等                                  │ │
│  └──────────────────────────────────────────────────────────────┘ │
│  • 所有工具在 Docker/gVisor 中执行                                │
│  • 资源限制（CPU、内存、磁盘、网络）                                │
│  • 审计日志 + 权限分级                                            │
│  • 成本：高  |  防御范围：全面                                     │
└──────────────────────────────────────────────────────────────────┘

升级决策：从内向外逐层添加，根据威胁模型决定止于哪一层。
  - 个人脚本/学习项目：第 1 层即可
  - 团队内部工具：至少到第 2 层
  - 面向用户的 Agent 服务：必须到第 3 层
```

### 6.2 当前实现的局限性

**黑名单的问题**：

```python
dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
if any(d in command for d in dangerous):
```

- 子字符串匹配可能误杀（如 `"echo 'use sudo carefully'"` 包含 `sudo`）
- 无法穷举所有危险命令
- 可以通过编码绕过（如 `base64 -d | bash`）

**路径沙盒的边界**：

- 不保护符号链接指向的文件（`resolve()` 会跟随符号链接，所以实际上是安全的）
- 不保护同名文件的竞争条件（TOCTOU 问题）
- `Path.resolve()` 在某些系统上行为可能不同（如不存在的路径）

### 6.3 生产级改进方向

| 改进 | 说明 | 复杂度 |
|------|------|--------|
| 白名单模式 | 只允许预定义的安全命令 | 低 |
| 命令解析 | 解析命令结构而非字符串匹配 | 中 |
| 审计日志 | 记录所有工具调用及结果 | 低 |
| 权限分级 | 读取无需确认，写入需确认 | 中 |
| 资源限制 | 限制单次调用的 CPU/内存/时间 | 中 |
| 容器隔离 | 在 Docker 中执行所有操作 | 高 |
| 用户确认钩子 | 危险操作前询问用户 | 低 |

---

## 7. 添加新工具的流程

### 7.1 标准三步流程

添加一个新工具只需三步，循环代码完全不变。每步完成后打勾确认：

```
步骤 1：定义 Schema          步骤 2：实现 Handler         步骤 3：注册到分发器
┌─────────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐
│ TOOLS.append({       │    │ def run_grep(...):   │    │ TOOL_HANDLERS[       │
│   "name": "...",     │    │     try:             │    │   "grep_search"      │
│   "description": …,  │    │       safe_path()    │    │ ] = lambda **kw: …   │
│   "input_schema": …  │    │       # 核心逻辑      │    │                      │
│ })                   │    │     except:          │    └─────────────────────┘
└─────────────────────┘    │       return "Error"  │
                            │ })                   │
  ☐ name 与 handler 键一致   └─────────────────────┘    ☐ lambda 提取的参数
  ☐ required 字段完整                                    与 schema properties 对应
  ☐ description 清晰       ☐ 使用 safe_path()
                            ☐ try/except 包裹
                            ☐ 输出截断 [:50000]
```

**验证公式**：`TOOLS[i]["name"] == TOOL_HANDLERS 的某个 key`，且 lambda 中提取的参数恰好是 schema 中声明的 `required` 加上可选参数。

```python
# 步骤 1：定义工具 schema
TOOLS.append({
    "name": "grep_search",
    "description": "Search for text in files.",
    "input_schema": {
        "type": "object",
        "properties": {
            "pattern": {"type": "string"},
            "path": {"type": "string"}
        },
        "required": ["pattern", "path"]
    }
})

# 步骤 2：实现处理函数
def run_grep(pattern: str, path: str) -> str:
    try:
        fp = safe_path(path)
        if not fp.exists():
            return f"Error: File not found: {path}"
        content = fp.read_text()
        lines = content.splitlines()
        matches = [f"{i+1}: {line}" for i, line in enumerate(lines)
                   if pattern in line]
        return "\n".join(matches)[:50000]
    except Exception as e:
        return f"Error: {e}"

# 步骤 3：注册到分发器
TOOL_HANDLERS["grep_search"] = lambda **kw: run_grep(kw["pattern"], kw["path"])
```

**循环代码不需要修改！**

### 7.2 工具设计的最佳实践

| 原则 | 说明 | 示例 |
|------|------|------|
| **单一职责** | 每个工具只做一件事 | `read_file` 和 `write_file` 分开 |
| **幂等性** | 重复执行产生相同结果 | `read_file` 可以多次调用 |
| **明确错误** | 失败时返回清晰的错误信息 | `"Error: Text not found"` 而非空字符串 |
| **限制输出** | 防止单个工具消耗过多上下文 | `[:50000]` 截断 |
| **参数验证** | 使用 `required` 字段声明必需参数 | schema 中的 `required` 数组 |
| **路径安全** | 所有文件路径经过 `safe_path` | 防止路径逃逸 |
| **异常兜底** | 用 try/except 包裹整个函数 | 确保不会因异常中断循环 |

---

## 8. 工具 vs 命令

### 8.1 何时使用专用工具，何时使用 bash

| 场景 | 推荐 | 原因 |
|------|------|------|
| 读取文件内容 | `read_file` | 更安全，有大小限制，经过路径沙盒 |
| 写入文件 | `write_file` | 防止路径逃逸，自动创建目录 |
| 精确编辑文件 | `edit_file` | 比 `sed` 更可靠，不受特殊字符影响 |
| 列出目录 | `bash("ls")` | 简单命令，bash 更灵活 |
| 复杂管道 | `bash("cat file \| grep x \| wc -l")` | bash 天然擅长管道操作 |
| 安装依赖 | `bash("pip install ...")` | 系统命令必须通过 bash |

### 8.2 安全边界

```python
# 安全：在工具层面控制
def run_read(path):
    path = safe_path(path)  # 强制在 WORKDIR 内
    return path.read_text()

# 不安全：通过 bash 操作文件
bash("cat /etc/passwd")  # 黑名单无法拦截所有敏感文件路径
```

**原则**：涉及文件内容的操作使用专用工具（有沙盒保护），涉及系统命令的操作使用 bash。

---

## 9. 调试工具系统

### 9.1 源码中已内置的调试

```python
print(f"> {block.name}: {output[:200]}")  # 打印工具名和前 200 字符结果
```

### 9.2 打印完整的工具调用

```python
for block in response.content:
    if block.type == "tool_use":
        print(f"[Tool] {block.name}({block.input})")
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input)
        print(f"[Result] {output[:100]}...")
```

### 9.3 验证工具定义与处理器的一致性

```python
def validate_tools(tools, handlers):
    """验证工具定义与处理器一一对应"""
    tool_names = {t["name"] for t in tools}
    handler_names = set(handlers.keys())
    if tool_names != handler_names:
        missing_handlers = tool_names - handler_names
        missing_tools = handler_names - tool_names
        if missing_handlers:
            print(f"Warning: Tools without handlers: {missing_handlers}")
        if missing_tools:
            print(f"Warning: Handlers without tools: {missing_tools}")
    else:
        print("All tools have matching handlers")
```

---

## 10. 常见问题

### Q1：为什么使用 lambda 而不是直接引用函数？

```python
# 方式 A：直接引用
TOOL_HANDLERS = {"read_file": run_read}

# 方式 B：lambda 包装（源码使用的方式）
TOOL_HANDLERS = {"read_file": lambda **kw: run_read(kw["path"], kw.get("limit"))}
```

**答案**：Lambda 提供了参数适配层：
1. 使用 `kw.get("limit")` 处理可选参数的默认值
2. 如果 `block.input` 包含额外参数，lambda 不会传递给函数（避免 `TypeError`）
3. 显式声明每个工具需要的参数，起到自文档化作用
4. 当函数签名与 schema 参数不完全匹配时，lambda 提供了灵活的映射能力

### Q2：如何处理工具执行错误？

源码中每个工具函数内部都使用 try/except 捕获异常：

```python
def run_read(path: str, limit: int = None) -> str:
    try:
        # ... 正常逻辑
    except Exception as e:
        return f"Error: {e}"
```

**最佳实践**：在处理函数内部捕获异常，返回友好的错误信息字符串。错误信息会作为 `tool_result` 返回给 LLM，LLM 看到错误后会自行决定如何处理（如换一种方式、修改参数等）。不要让异常传播到循环层，否则会中断整个 Agent。

### Q3：工具可以调用其他工具吗？

```python
def run_composite_operation(path: str) -> str:
    content = run_read(path)      # 调用另一个工具函数
    result = run_write(f"{path}.bak", content)
    return result
```

**答案**：技术上可以，因为工具函数就是普通 Python 函数。但不推荐这种做法，原因：
1. 违反单一职责原则
2. 增加调试难度
3. 错误传播链更长
4. LLM 可以通过多轮工具调用自己组合操作，不需要工具内部嵌套

### Q4：`safe_path` 能防御所有路径攻击吗？

**答案**：不能完全防御。`resolve()` + `is_relative_to()` 可以防御：
- `../` 路径穿越
- 混合 `./` 和 `../` 的混淆路径
- 绝对路径注入

但不能防御：
- 符号链接竞态条件（TOCTOU）
- 不存在路径的 `resolve()` 行为在不同 OS 上可能不同
- 如果 `WORKDIR` 本身是符号链接，`resolve()` 会跟随它

---

## 调试清单

当你遇到工具系统相关的问题时，按此清单逐项排查：

| 症状 | 可能原因 | 排查方法 | 修复方案 |
|------|---------|---------|---------|
| 工具调用返回 `"Unknown tool: ..."` | `TOOLS` 中的 `name` 与 `TOOL_HANDLERS` 的 key 不一致 | 打印 `set(t["name"] for t in TOOLS)` 与 `set(TOOL_HANDLERS.keys())` 对比 | 统一 `name` 和 key 的拼写 |
| 工具调用抛出 `TypeError` | lambda 中的参数提取与 `block.input` 实际内容不匹配 | 打印 `block.input` 查看实际参数 | 修正 lambda 提取逻辑，或用 `kw.get()` 处理可选参数 |
| 工具调用抛出 `KeyError` | schema 声明为 `required` 的参数未被 LLM 传递 | 检查 schema 的 `required` 列表与 LLM 实际输出 | 改善工具 `description` 引导 LLM 传参，或在 lambda 中添加默认值 |
| `ValueError: Path escapes workspace` | LLM 尝试访问工作目录之外的路径 | 打印 `safe_path()` 的输入与 `WORKDIR` | 修改 system prompt 限制操作范围，或在 handler 中提供相对路径转换 |
| 工具返回空字符串或 `"(no output)"` | 工具执行成功但无输出，或异常被吞掉 | 在 handler 中添加 `print` 打印中间状态 | 检查是否需要返回更有意义的结果信息 |
| 工具调用卡住不返回 | bash 命令执行时间超过 120s 超时 | 检查命令是否需要交互式输入（如 `sudo` 密码） | 增加超时时间，或避免需要交互的命令 |
| LLM 反复调用同一工具 | 工具返回的错误信息不够清晰，LLM 无法理解问题 | 查看工具返回的错误文本 | 改善错误信息，包含具体的失败原因和建议的修正方式 |

> **快速定位口诀**：Unknown 看注册，TypeError 看参数，KeyError 看 required，空输出加日志。

---

## 实战应用：为你的项目设计工具系统

掌握了分发器模式和安全机制后，我们来思考如何为真实业务场景设计一套完整的工具系统。

### 三大真实场景

**场景 1：数据库操作的 Agent**

- **背景**：日常需要查询数据、分析慢查询、偶尔修改配置表。
- **工具设计**：
  - `read_sql(query)` -- 执行 SELECT，返回 JSON 格式结果（只读，安全）
  - `write_sql(query)` -- 执行 INSERT/UPDATE/DELETE，需要二次确认
  - `explain_query(query)` -- 返回执行计划，帮助优化慢查询
- **安全考量**：`write_sql` 可通过白名单限制只允许操作特定表，`read_sql` 添加结果行数上限防止拉取全表。

**场景 2：云资源管理的 Agent**

- **背景**：需要在多云环境中查看实例状态、部署服务、检查健康度。
- **工具设计**：
  - `list_instances(provider, region)` -- 列出计算实例
  - `deploy_service(provider, config_json)` -- 部署新服务（需要确认钩子）
  - `check_status(service_name)` -- 查询服务健康状态
- **安全考量**：部署操作属于高风险，建议在 `SYSTEM` 提示中要求 LLM 先展示计划再执行，并在 handler 中添加人工审批环节。

**场景 3：文档维护的 Agent**

- **背景**：技术文档分散在多个仓库，需要扫描一致性、检查链接有效性、维护索引。
- **工具设计**：
  - `scan_docs(directory)` -- 扫描目录下所有 Markdown 文件，返回结构和元信息
  - `check_links(path)` -- 检查文件中所有超链接是否有效
  - `update_index(changes_json)` -- 根据变更更新文档索引文件
- **安全考量**：`scan_docs` 和 `check_links` 为只读操作风险低；`update_index` 需要路径沙盒限制在文档目录内。

### 工具设计 Checklist

在为项目设计每个工具时，逐项检查下表：

| 检查项 | 说明 | 通过？ |
|--------|------|--------|
| 单一职责 | 工具是否做且只做一件事？不要把"读取并分析"塞进一个工具 | ☐ |
| JSON Schema | 参数是否有完整的 JSON Schema 定义（含 `type`、`required`）？ | ☐ |
| 安全检查 | 路径类参数是否经过沙盒校验？输入是否防注入？ | ☐ |
| 错误信息友好 | 错误信息是否足够让 LLM 理解问题并自动修正？避免返回原始异常栈 | ☐ |
| 输出截断 | 返回内容是否限制在合理长度（建议 50000 字符以内）？ | ☐ |
| 异常兜底 | 是否有 try/except 包裹，确保不会中断 Agent 循环？ | ☐ |
| 可观测性 | 工具调用是否有日志输出，便于调试？ | ☐ |
| 幂等安全 | 重复调用是否安全（至少不会造成数据损坏）？ | ☐ |

### 与外部框架对比

本课程使用的分发器模式并非独创，主流 Agent 框架的核心思想完全一致，只是注册方式不同：

| 方面 | 本课程分发器 | LangChain Tools | CrewAI Tools |
|------|-------------|-----------------|--------------|
| 注册方式 | 字典 `{"name": lambda ...}` | `@tool` 装饰器 | 类继承 `BaseTool` |
| 核心思想 | `name -> handler` 映射 | 相同 | 相同 |
| Schema 来源 | 手写 JSON Schema | 从函数签名自动推断 | 从类属性定义 |
| 学习曲线 | 极低（就是 Python 字典） | 低 | 中 |
| 灵活性 | 完全可控 | 框架约束 | 框架约束 |

**核心结论**：无论框架如何包装，工具系统的本质都是"名称到处理函数的映射"。理解了本课程的分发器，你就能快速掌握任何框架的工具机制。

---

## 11. 小结

### 11.1 核心要点

| 要点 | 说明 |
|------|------|
| **分发器模式** | 工具名称到 lambda 处理函数的映射字典 |
| **Lambda + `**kw`** | 显式参数提取，提供适配层和默认值控制 |
| **添加工具 = 添加处理器** | 循环结构完全不变 |
| **路径沙盒** | `safe_path()` 使用 `resolve()` + `is_relative_to()` 防止路径逃逸 |
| **职责分离** | 工具定义（schema）与工具执行（handler）分离 |
| **错误处理** | 每个工具内部 try/except，返回字符串错误信息 |

### 11.2 代码模板

```python
# 1. 定义工具 schema
TOOLS = [{"name": "...", "description": "...", "input_schema": {...}}, ...]

# 2. 实现处理函数（带 safe_path + try/except）
def run_tool_name(param1, param2):
    try:
        # ... 安全逻辑
    except Exception as e:
        return f"Error: {e}"

# 3. 注册到分发器（lambda 包装参数适配）
TOOL_HANDLERS = {"tool_name": lambda **kw: run_tool_name(kw["param1"], kw.get("param2"))}

# 4. 循环中使用分发器
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
```

### 11.3 关键洞察

```
工具系统的设计目标：
1. 添加工具不修改核心循环
2. 每个工具独立且可测试
3. 工具定义与执行分离
4. 安全控制在工具层面实现

从 s01 到 s02：
- 循环代码只改了一行（硬编码 -> 分发器查找）
- 安全性从黑名单升级到路径沙盒
- 工具从 1 个扩展到 4 个
- 但核心循环骨架完全不变
```

---

## 12. 练习

### 练习 1：添加 `list_files` 工具

实现一个工具，列出目录中的所有文件，支持递归选项。注意使用 `safe_path` 确保安全。

### 练习 2：添加 `delete_file` 工具

实现删除文件的工具。思考：是否需要额外的安全机制（如确认提示、回收站）？

### 练习 3：改进错误处理

修改工具处理函数，使其返回结构化的错误信息（包含错误类型和建议操作）。

### 练习 4：实现命令白名单

将 `run_bash` 的黑名单模式改为白名单模式，只允许预定义的安全命令（如 `ls`、`cat`、`grep`、`find`、`wc`）。

### 练习 5：验证分发器一致性

编写一个函数，检查 `TOOLS` 中的每个工具名称是否都在 `TOOL_HANDLERS` 中有对应的处理器，反之亦然。

---

## 13. 下一步

现在你的 Agent 可以安全地操作文件了。但它还缺乏规划能力 -- 面对多步骤任务，它可能会迷失方向。

在 [s03：任务规划](./s03-todo-write.md) 中，我们将学习：
- 如何让 Agent 先规划再执行
- TodoManager 的实现
- 提醒机制的工作原理

---

< 上一章：[s01-Agent 循环](./s01-the-agent-loop.md) | [目录](./index.md) | 下一章：[s03-任务规划](./s03-todo-write.md) >
