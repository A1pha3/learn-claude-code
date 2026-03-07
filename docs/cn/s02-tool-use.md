# s02：工具系统 —— 添加工具的正确方式

> **核心口号**：*"添加工具 = 添加处理器"*

> **学习目标**：掌握多工具系统的设计模式，理解分发器的作用，实现安全的文件操作

---

## 学习目标

完成本章后，你将能够：

1. **理解分发器模式** —— 为什么不用 if/elif 链
2. **实现安全的文件操作** —— 路径沙盒和权限控制
3. **掌握工具注册机制** —— 添加工具不影响循环结构
4. **理解职责分离** —— 工具定义与工具执行的分离

---

## 1. 从一个工具到多个工具

### 1.1 回顾 s01

在 s01 中，我们只有一个 `bash` 工具：

```python
# s01 的代码
for block in response.content:
    if block.type == "tool_use":
        command = block.input["command"]  # 硬编码
        output = os.popen(command).read()  # 直接执行
```

**问题**：如果要添加 `read_file`、`write_file` 等工具，代码会变成什么？

### 1.2 错误的做法：if/elif 链

```python
# ❌ 不推荐：if/elif 链
for block in response.content:
    if block.type == "tool_use":
        if block.name == "bash":
            command = block.input["command"]
            output = os.popen(command).read()
        elif block.name == "read_file":
            path = block.input["path"]
            output = open(path).read()
        elif block.name == "write_file":
            path = block.input["path"]
            content = block.input["content"]
            open(path, "w").write(content)
        elif block.name == "edit_file":
            # ... 更多代码
        # 每添加一个工具，都要修改这里！
```

**问题**：
1. 代码臃肿，难以维护
2. 添加工具需要修改核心逻辑
3. 违反开闭原则（对扩展开放，对修改封闭）

### 1.3 正确的做法：分发器（Dispatcher）

```python
# ✅ 推荐：分发器模式
TOOL_HANDLERS = {
    "bash": lambda **kw: run_bash(kw["command"]),
    "read_file": lambda **kw: run_read(kw["path"]),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file": lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}

for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        if handler:
            output = handler(**block.input)
        else:
            output = f"Unknown tool: {block.name}"
```

**优点**：
1. 添加工具只需在字典中添加一项
2. 核心循环逻辑不变
3. 工具处理函数可以独立测试

---

## 2. 分发器模式详解

### 2.1 什么是分发器

分发器是一个**工具名称到处理函数的映射**：

```python
# 工具名称     ->  处理函数
#      ↓                ↓
TOOL_HANDLERS = {
    "bash":       run_bash,
    "read_file":  run_read,
    "write_file": run_write,
}
```

### 2.2 分发器的工作流程

```
LLM 响应：tool_use("read_file", {"path": "test.txt"})
                    │
                    ▼
           分发器查找：TOOL_HANDLERS["read_file"]
                    │
                    ▼
           找到函数：run_read
                    │
                    ▼
           调用函数：run_read(path="test.txt")
                    │
                    ▼
           返回结果：文件内容
```

### 2.3 为什么使用 `**kwargs`

```python
def run_read(path: str, limit: int = None) -> str:
    # 函数定义

handler = TOOL_HANDLERS.get("read_file")
output = handler(**block.input)  # 展开字典为关键字参数
```

**工作原理**：

```python
block.input = {"path": "test.txt", "limit": 100}

# **block.input 展开为：
handler(path="test.txt", limit=100)

# 如果 block.input 有额外参数：
block.input = {"path": "test.txt", "limit": 100, "extra": "value"}
# handler 会忽略 extra（因为函数没有这个参数）
```

**优点**：
1. 灵活：工具可以接受不同的参数组合
2. 向后兼容：添加新参数不会破坏旧代码
3. 简洁：不需要手动提取每个参数

### 2.4 Lambda vs 函数引用

```python
# 方式 1：直接引用函数
TOOL_HANDLERS = {
    "read_file": run_read,  # 直接引用
}

# 方式 2：使用 lambda 调整参数
TOOL_HANDLERS = {
    "read_file": lambda **kw: run_read(kw["path"], kw.get("limit")),
}
```

**何时使用哪种**：

| 场景 | 推荐方式 | 原因 |
|------|----------|------|
| 函数签名与工具 schema 完全匹配 | 直接引用 | 更简洁 |
| 需要调整参数顺序或默认值 | lambda | 灵活适配 |
| 需要额外的错误处理 | lambda | 可以包装逻辑 |

---

## 3. 实现安全的文件操作

### 3.1 为什么不能直接用 bash

s01 的 `bash` 工具可以执行任何命令，这带来了严重的安全问题：

```python
# 危险操作
bash("rm -rf /")           # 删除整个文件系统
bash("cat /etc/passwd")    # 读取敏感文件
bash("cd ../../; rm important.txt")  # 跳出工作目录
```

**解决方案**：使用专门的文件操作工具，在工具层面实现安全控制。

### 3.2 路径沙盒

路径沙盒确保所有文件操作都在指定目录内：

```python
from pathlib import Path

WORKDIR = Path("/workspace").resolve()

def safe_path(p: str) -> Path:
    """确保路径在 WORKDIR 内"""
    path = (WORKDIR / p).resolve()  # 解析为绝对路径

    # 检查路径是否逃逸
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"路径逃逸工作目录: {p}")

    return path
```

**示例**：

```python
safe_path("test.txt")           # ✅ /workspace/test.txt
safe_path("./subdir/file.txt")  # ✅ /workspace/subdir/file.txt
safe_path("../escape.txt")      # ❌ ValueError: 路径逃逸
safe_path("/etc/passwd")        # ❌ ValueError: 路径逃逸
```

### 3.3 读取文件

```python
def run_read(path: str, limit: int = None) -> str:
    """安全地读取文件"""
    path = safe_path(path)  # 安全检查

    if not path.exists():
        return f"错误：文件不存在: {path}"

    text = path.read_text(encoding="utf-8")
    lines = text.splitlines()

    # 可选：限制读取的行数
    if limit and limit < len(lines):
        lines = lines[:limit]
        return "\n".join(lines) + f"\n...（共 {len(lines)} 行，已截断）"

    # 限制返回的字符数
    return text[:50000]
```

**设计要点**：
1. 使用 `safe_path` 防止路径逃逸
2. 检查文件是否存在
3. 限制输出大小（行数和字符数）
4. 处理编码错误

### 3.4 写入文件

```python
def run_write(path: str, content: str) -> str:
    """安全地写入文件"""
    path = safe_path(path)

    # 创建父目录
    path.parent.mkdir(parents=True, exist_ok=True)

    # 写入文件
    path.write_text(content, encoding="utf-8")

    return f"已写入 {len(content)} 字符到 {path}"
```

### 3.5 编辑文件

```python
def run_edit(path: str, old_text: str, new_text: str) -> str:
    """通过查找替换编辑文件"""
    path = safe_path(path)

    if not path.exists():
        return f"错误：文件不存在: {path}"

    content = path.read_text(encoding="utf-8")

    if old_text not in content:
        return f"错误：未找到要替换的文本"

    # 执行替换
    new_content = content.replace(old_text, new_text, 1)  # 只替换第一个
    path.write_text(new_content, encoding="utf-8")

    return f"已替换文本"
```

**为什么不用 sed？**

1. `sed` 在特殊字符上容易出错
2. Python 字符串操作更可靠
3. 可以精确控制替换次数

---

## 4. 完整的工具系统实现

### 4.1 工具定义

```python
TOOLS = [
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
    },
    {
        "name": "read_file",
        "description": "读取文件内容",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "文件路径（相对于工作目录）"
                },
                "limit": {
                    "type": "number",
                    "description": "可选：限制读取的行数"
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
        "name": "edit_file",
        "description": "通过查找替换编辑文件",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "old_text": {"type": "string"},
                "new_text": {"type": "string"}
            },
            "required": ["path", "old_text", "new_text"]
        }
    },
]
```

### 4.2 工具处理器

```python
from pathlib import Path
import subprocess
import os

WORKDIR = Path("/workspace").resolve()

def safe_path(p: str) -> Path:
    """确保路径在 WORKDIR 内"""
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"路径逃逸工作目录: {p}")
    return path

def run_bash(command: str) -> str:
    """执行 Bash 命令"""
    result = subprocess.run(
        command,
        shell=True,
        cwd=WORKDIR,
        capture_output=True,
        text=True,
        timeout=30
    )
    return (result.stdout + result.stderr)[:50000]

def run_read(path: str, limit: int = None) -> str:
    """读取文件"""
    path = safe_path(path)
    if not path.exists():
        return f"错误：文件不存在: {path}"
    text = path.read_text(encoding="utf-8")
    lines = text.splitlines()
    if limit and limit < len(lines):
        lines = lines[:limit]
    return "\n".join(lines)[:50000]

def run_write(path: str, content: str) -> str:
    """写入文件"""
    path = safe_path(path)
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(content, encoding="utf-8")
    return f"已写入 {len(content)} 字符到 {path}"

def run_edit(path: str, old_text: str, new_text: str) -> str:
    """编辑文件"""
    path = safe_path(path)
    if not path.exists():
        return f"错误：文件不存在: {path}"
    content = path.read_text(encoding="utf-8")
    if old_text not in content:
        return f"错误：未找到要替换的文本"
    new_content = content.replace(old_text, new_text, 1)
    path.write_text(new_content, encoding="utf-8")
    return f"已替换文本"

# 分发器
TOOL_HANDLERS = {
    "run_bash": lambda **kw: run_bash(kw["command"]),
    "read_file": lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file": lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}
```

### 4.3 集成到循环中

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]

    while True:
        response = client.messages.create(
            model=MODEL,
            messages=messages,
            tools=TOOLS,  # 工具定义
            max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            break

        results = []
        for block in response.content:
            if block.type == "tool_use":
                # 使用分发器
                handler = TOOL_HANDLERS.get(block.name)
                if handler:
                    try:
                        output = handler(**block.input)
                    except Exception as e:
                        output = f"错误: {str(e)}"
                else:
                    output = f"未知工具: {block.name}"

                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output)[:50000],
                })

        messages.append({"role": "user", "content": results})

    return response.content
```

**关键点**：循环结构与 s01 完全相同，只是：
- `TOOLS` 变量有更多工具定义
- 使用分发器查找工具处理器

---

## 5. 添加新工具的流程

### 5.1 标准流程

添加新工具只需要三步：

```python
# 步骤 1：定义工具 schema
TOOLS.append({
    "name": "grep_search",
    "description": "在文件中搜索文本",
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
    path = safe_path(path)
    if not path.exists():
        return f"文件不存在: {path}"
    content = path.read_text()
    lines = content.splitlines()
    matches = [f"{i+1}: {line}" for i, line in enumerate(lines)
               if pattern in line]
    return "\n".join(matches)

# 步骤 3：注册到分发器
TOOL_HANDLERS["grep_search"] = lambda **kw: run_grep(kw["pattern"], kw["path"])
```

**循环代码不需要修改！**

### 5.2 工具设计的最佳实践

| 原则 | 说明 | 示例 |
|------|------|------|
| **单一职责** | 每个工具只做一件事 | `read_file` 和 `write_file` 分开 |
| **幂等性** | 重复执行产生相同结果 | `read_file` 可以多次调用 |
| **明确错误** | 失败时返回清晰的错误信息 | "文件不存在" 而非空字符串 |
| **限制输出** | 防止单个工具消耗过多上下文 | 截断过大的输出 |
| **参数验证** | 检查必需参数 | 使用 `required` 字段 |

---

## 6. 工具 vs 命令

### 6.1 何时使用专用工具，何时使用 bash

| 场景 | 推荐 | 原因 |
|------|------|------|
| 读取文件内容 | `read_file` | 更安全，有大小限制 |
| 写入文件 | `write_file` | 防止路径逃逸 |
| 列出目录 | `bash("ls")` | 简单命令可以用 bash |
| 复杂管道 | `bash("cat file \| grep x \| wc -l")` | bash 更灵活 |

### 6.2 安全边界

```python
# ✅ 安全：在工具层面控制
def run_read(path):
    path = safe_path(path)  # 强制在 WORKDIR 内
    return path.read_text()

# ❌ 不安全：依赖系统权限
def run_bash(command):
    return subprocess.run(command, shell=True)  # 可能读取 /etc/passwd
```

**生产建议**：通过白名单限制 bash 命令，或者只允许特定命令（如 `ls`、`grep`、`find`）。

---

## 7. 调试工具系统

### 7.1 打印工具调用

```python
for block in response.content:
    if block.type == "tool_use":
        print(f"[工具] {block.name}({block.input})")
```

### 7.2 打印工具结果

```python
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input)
        print(f"[结果] {output[:100]}...")
```

### 7.3 验证工具定义

```python
def validate_tools(tools):
    """验证工具定义"""
    for tool in tools:
        assert "name" in tool
        assert "description" in tool
        assert "input_schema" in tool
        assert tool["input_schema"]["type"] == "object"
        print(f"✅ {tool['name']}")
```

---

## 8. 与 s01 的对比

| 方面 | s01 | s02 |
|------|-----|-----|
| 工具数量 | 1 (bash) | 4 (bash, read, write, edit) |
| 工具查找 | 直接调用 | 分发器字典 |
| 路径安全 | 无 | `safe_path()` 沙盒 |
| 文件操作 | 通过 bash | 专用工具 |
| 循环结构 | `while stop_reason` | 相同（未改变） |

**核心洞察**：从 1 个工具到 N 个工具，循环结构保持不变。

---

## 9. 常见问题

### Q1：为什么使用 lambda 而不是直接引用函数？

```python
# 方式 A：直接引用
TOOL_HANDLERS = {"read_file": run_read}

# 方式 B：lambda
TOOL_HANDLERS = {"read_file": lambda **kw: run_read(kw["path"], kw.get("limit"))}
```

**答案**：当函数签名与工具 schema 的参数完全匹配时，直接引用更简洁。但 lambda 提供了灵活性，可以调整参数顺序或添加默认值。

### Q2：如何处理工具执行错误？

```python
try:
    output = handler(**block.input)
except Exception as e:
    output = f"工具执行失败: {str(e)}"
```

**最佳实践**：在处理函数内部捕获异常，返回友好的错误信息，而不是让异常传播到循环。

### Q3：工具可以调用其他工具吗？

```python
def run_composite_operation(path: str) -> str:
    # 在工具内部调用其他工具？
    content = run_read(path)  # 调用另一个工具
    result = run_write(f"{path}.bak", content)
    return result
```

**答案**：技术上可以，但不推荐。工具应该保持简单和独立。复杂逻辑应该由 LLM 通过多次工具调用来完成。

---

## 10. 小结

### 10.1 核心要点

| 要点 | 说明 |
|------|------|
| **分发器模式** | 工具名称到处理函数的映射 |
| **添加工具** = 添加处理器 | 循环结构不变 |
| **路径沙盒** | `safe_path()` 防止路径逃逸 |
| **职责分离** | 工具定义与工具执行分离 |

### 10.2 代码模板

```python
# 1. 定义工具
TOOLS = [{"name": "...", "input_schema": {...}}, ...]

# 2. 实现处理函数
def run_tool_name(param1, param2): ...

# 3. 注册到分发器
TOOL_HANDLERS = {"tool_name": lambda **kw: run_tool_name(**kw)}

# 4. 循环中使用
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input)
```

### 10.3 关键洞察

```
工具系统的设计目标：
1. 添加工具不修改核心循环
2. 每个工具独立且可测试
3. 工具定义与执行分离
4. 安全控制在工具层面实现
```

---

## 11. 练习

### 练习 1：添加 `list_files` 工具

实现一个工具，列出目录中的所有文件，支持递归选项。

### 练习 2：添加 `delete_file` 工具

实现删除文件的工具，确保有安全确认机制。

### 练习 3：改进错误处理

修改工具处理函数，使其返回结构化的错误信息（包含错误类型和建议）。

### 练习 4：工具白名单

实现一个命令白名单系统，`bash` 工具只能执行白名单中的命令。

---

## 12. 下一步

现在你的 Agent 可以安全地操作文件了。但它还缺乏规划能力——面对多步骤任务，它可能会迷失方向。

在 [s03：任务规划](./s03-todo-write.md) 中，我们将学习：
- 如何让 Agent 先规划再执行
- TodoManager 的实现
- 提醒机制的工作原理

---

< 上一章：[s01-Agent 循环](./s01-the-agent-loop.md) | [目录](./index.md) | 下一章：[s03-任务规划](./s03-todo-write.md) >
