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

**源码选择 lambda 的原因**：

1. **参数默认值控制**：`kw.get("limit")` 而非 `kw["limit"]`，当 LLM 不传 `limit` 时返回 `None`，而 `run_read(path: str, limit: int = None)` 正好接收 `None` 作为默认值
2. **参数名映射**：如果工具 schema 的参数名与函数参数名不同，lambda 可以做映射
3. **防御性编程**：如果 `block.input` 包含函数不认识的额外参数，lambda 通过显式提取可以忽略它们
4. **接口隔离**：每个 lambda 明确声明了该工具需要哪些参数，自文档化

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

---

## 6. 工具安全设计的生产级考量

### 6.1 安全防护的三个层次

```
第 1 层：黑名单拦截（s01 实现）
    - 拦截已知危险命令
    - 简单但不够全面

第 2 层：路径沙盒（s02 实现）
    - 限制文件操作范围
    - 使用 resolve() + is_relative_to() 防止逃逸

第 3 层：生产环境还需补充
    - 资源限制（CPU、内存、磁盘）
    - 审计日志（记录所有工具调用）
    - 权限分级（不同操作需要不同权限）
    - 沙箱隔离（容器/虚拟机级别）
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
