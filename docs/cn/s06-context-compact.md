# s06：上下文压缩 —— 实现无限会话

> **核心口号**：*"上下文会填满，需要腾出空间"*

> **学习目标**：理解上下文窗口的限制，掌握三层压缩策略，学会实现可持续的长对话

---

## 学习目标

完成本章后，你将能够：

1. **理解上下文饱和问题** —— 为什么长对话会"变笨"
2. **掌握三层压缩策略** —— 微压缩、自动压缩、手动压缩
3. **实现转录存档** —— 保存完整历史到磁盘
4. **设计压缩触发机制** —— 何时触发自动压缩

---

## 1. 问题的本质

### 1.1 上下文饱和过程

想象一个典型的编程会话：

```
轮次 1（100 tokens）：
  用户：帮我修复登录功能的 bug

轮次 2（5000 tokens）：
  Agent：让我先看看代码
        read_file("auth.py") → 返回 4000 tokens
        read_file("database.py") → 返回 3000 tokens
        read_file("api.py") → 返回 3500 tokens

轮次 3（3000 tokens）：
  Agent：找到问题了，让我修复
        edit_file(...) → 返回 100 tokens

轮次 4（2000 tokens）：
  Agent：验证修复
        run_bash("pytest tests/test_auth.py") → 返回 1800 tokens

轮次 5（1500 tokens）：
  Agent：测试通过！

总计：~12,000 tokens
```

**问题**：对话越长，上下文越大，最终可能超出窗口限制（200,000 tokens）。

### 1.2 饱和的后果

```
上下文窗口（200,000 tokens）：

┌─────────────────────────────────────┐
│ 系统提示（1,000 tokens）████        │
│ 用户问题（500 tokens）██            │
│ 工具调用 1（100 tokens）             │
│ 工具结果 1（4,000 tokens）██████████│
│ 工具调用 2（100 tokens）             │
│ 工具结果 2（3,000 tokens）████████  │
│ ... 40 轮对话后 ...                 │
│                                     │
│ [系统提示被挤出]                    │
│ [原始问题被遗忘]                    │
│ [重要决策丢失]                      │
└─────────────────────────────────────┘
```

**后果**：
1. 系统提示被遗忘 → Agent 偏离原始角色
2. 用户问题被淹没 → Agent 忘记最终目标
3. Token 预算耗尽 → 无法继续对话

### 1.3 解决方案：三层压缩

```
策略 1：微压缩（每轮执行）
  用占位符替换旧的工具结果
  "读取了 auth.py" ← [Previous: used read_file]

策略 2：自动压缩（触发时执行）
  保存完整记录到磁盘
  用摘要替换整个对话历史

策略 3：手动压缩（Agent 主动调用）
  Agent 决定现在需要压缩
  调用 compact 工具
```

---

## 2. 微压缩（Micro-compact）

### 2.1 思想

旧的工具结果不再有用，用占位符替换它们：

```python
# 压缩前：
{
    "type": "tool_result",
    "tool_use_id": "tool_123",
    "content": "def authenticate(user, password):\n    # 4000 lines of code\n..."
}

# 压缩后：
{
    "type": "tool_result",
    "tool_use_id": "tool_123",
    "content": "[Previous: used read_file on auth.py]"
}
```

### 2.2 实现

```python
KEEP_RECENT = 5  # 保留最近 5 个工具结果

def micro_compact(messages: list) -> list:
    """每轮都执行的微压缩"""
    tool_results = []

    # 收集所有工具结果的位置
    for i, msg in enumerate(messages):
        if msg["role"] == "user" and isinstance(msg.get("content"), list):
            for j, part in enumerate(msg["content"]):
                if isinstance(part, dict) and part.get("type") == "tool_result":
                    tool_results.append((i, j, part))

    # 如果工具结果不多，不需要压缩
    if len(tool_results) <= KEEP_RECENT:
        return messages

    # 压缩旧的工具结果（保留最近的几个）
    for i, j, part in tool_results[:-KEEP_RECENT]:
        content = part.get("content", "")
        # 只压缩长内容
        if len(content) > 100:
            # 尝试提取工具名
            tool_name = part.get("tool_use_id", "unknown")
            part["content"] = f"[Previous: tool result from {tool_name}]"

    return messages
```

### 2.3 集成到循环中

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]

    while True:
        # ========== 每轮都执行微压缩 ==========
        messages = micro_compact(messages)

        response = client.messages.create(
            model=MODEL,
            messages=messages,
            tools=TOOLS,
            max_tokens=8000,
        )
        # ... 其余逻辑
```

**效果**：每轮都清理旧的工具结果，保持上下文精简。

---

## 3. 自动压缩（Auto-compact）

### 3.1 思想

当 Token 估算超过阈值时，自动触发深度压缩：

```python
# 压缩前（50,000 tokens）：
messages = [
    {"role": "user", "content": "原始问题"},
    # ... 40 轮对话，大量的工具调用和结果 ...
]

# 压缩后（3,000 tokens）：
messages = [
    {"role": "user", "content": """
[会话已压缩]

摘要：
用户要求修复登录功能的 bug。
Agent 读取了 auth.py、database.py、api.py 等文件，
发现密码验证逻辑错误，已修复并通过测试。

原始问题已解决，所有测试通过。
"""},
    {"role": "assistant", "content": "了解，继续工作。"}
]
```

### 3.2 Token 估算

```python
import tiktoken

def estimate_tokens(messages: list) -> int:
    """估算消息列表的 Token 数量"""
    # 使用 cl100k_base 编码器（GPT-4/Claude 类似）
    encoder = tiktoken.get_encoding("cl100k_base")

    total = 0
    for msg in messages:
        # 角色名
        total += len(encoder.encode(msg["role"]))
        # 内容
        content = msg.get("content", "")
        if isinstance(content, list):
            for part in content:
                total += len(encoder.encode(str(part)))
        else:
            total += len(encoder.encode(content))

    return total
```

### 3.3 压缩实现

```python
import json
import time
from pathlib import Path

TRANSCRIPT_DIR = Path(".transcripts")
TRANSCRIPT_DIR.mkdir(exist_ok=True)

COMPACT_THRESHOLD = 50000  # Token 阈值

def auto_compact(messages: list) -> list:
    """自动深度压缩"""
    # 1. 保存完整记录到磁盘
    timestamp = int(time.time())
    transcript_path = TRANSCRIPT_DIR / f"transcript_{timestamp}.jsonl"

    with open(transcript_path, "w", encoding="utf-8") as f:
        for msg in messages:
            f.write(json.dumps(msg, ensure_ascii=False) + "\n")

    # 2. 请求 LLM 生成摘要
    summary_prompt = f"""请将以下对话历史压缩为简洁的摘要。

摘要应该包含：
1. 原始用户请求
2. 执行的主要操作
3. 当前的状态和结果
4. 任何待办事项

对话历史：
{json.dumps(messages, ensure_ascii=False)[:80000]}

请生成摘要："""

    summary_response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content": summary_prompt}],
        max_tokens=2000,
    )

    summary = summary_response.content[0].text

    # 3. 用摘要替换历史
    return [
        {
            "role": "user",
            "content": f"[会话已压缩于 {timestamp}]\n\n{summary}"
        },
        {
            "role": "assistant",
            "content": "了解。请继续当前工作。"
        }
    ]
```

### 3.4 触发机制

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]

    while True:
        # 微压缩
        messages = micro_compact(messages)

        # 检查是否需要自动压缩
        if estimate_tokens(messages) > COMPACT_THRESHOLD:
            print(f"[自动压缩] 估算 Token: {estimate_tokens(messages)}, 阈值: {COMPACT_THRESHOLD}")
            messages = auto_compact(messages)

        response = client.messages.create(...)
        # ... 其余逻辑
```

---

## 4. 手动压缩（Manual-compact）

### 4.1 思想

让 Agent 自己决定何时压缩：

```python
# Agent 可以主动调用
tool_use: compact()
```

### 4.2 工具定义

```python
TOOLS.append({
    "name": "compact",
    "description": "压缩当前对话历史。当上下文过长或不再需要详细历史时使用。",
    "input_schema": {
        "type": "object",
        "properties": {},
        "required": []
    }
})

TOOL_HANDLERS["compact"] = lambda **kw: auto_compact(messages)
```

### 4.3 Agent 的使用场景

```
LLM：我已经完成了第一个阶段的开发，现在进入第二阶段。
      详细的文件读取记录不再需要了。
      tool_use: compact()

工具结果：[会话已压缩]

LLM：继续第二阶段...
```

---

## 5. 转录存档

### 5.1 为什么要保存完整历史

压缩会丢失细节，但完整历史可能有价值：
- 调试：查看 Agent 的完整决策过程
- 审计：了解做了哪些修改
- 学习：分析 Agent 的行为模式

### 5.2 存储格式

使用 JSONL（每行一个 JSON 对象）：

```jsonl
{"role": "user", "content": "修复登录 bug"}
{"role": "assistant", "content": [{"type": "tool_use", ...}]}
{"role": "user", "content": [{"type": "tool_result", ...}]}
{"role": "assistant", "content": [{"type": "tool_use", ...}]}
...
```

### 5.3 恢复机制

```python
def load_transcript(transcript_path: Path) -> list:
    """从转录文件恢复消息"""
    messages = []
    with open(transcript_path, "r", encoding="utf-8") as f:
        for line in f:
            if line.strip():
                messages.append(json.loads(line))
    return messages
```

---

## 6. 三层压缩的协同

### 6.1 工作流程

```
每轮循环：
┌─────────────────────────────────────┐
│ 1. 微压缩（总是执行）                │
│    - 替换旧的工具结果                │
│    - 成本：~0                        │
└─────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ 2. Token 预算检查                    │
│    - 估算当前 Token 数               │
│    - 超过阈值？                      │
└─────────────────────────────────────┘
           │
           ├─ 否 ─→ 继续正常循环
           │
           └─ 是 ─▼
┌─────────────────────────────────────┐
│ 3. 自动压缩（触发时执行）            │
│    - 保存完整记录                    │
│    - 生成摘要                        │
│    - 替换历史                        │
└─────────────────────────────────────┘

随时（Agent 决定）：
┌─────────────────────────────────────┐
│ 手动压缩                             │
│ - Agent 调用 compact 工具            │
│ - 同自动压缩机制                     │
└─────────────────────────────────────┘
```

### 6.2 参数调优

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `KEEP_RECENT` | 5-10 | 微压缩保留的工具结果数 |
| `COMPACT_THRESHOLD` | 50000-100000 | 自动压缩的 Token 阈值 |
| `SUMMARY_TOKENS` | 2000-4000 | 摘要的最大 Token 数 |

### 6.3 效果对比

```
无压缩：
轮次 1-50：正常增长
轮次 51：超出窗口，无法继续

有微压缩：
轮次 1-100：缓慢增长（旧结果被替换）
轮次 101：超出窗口

有微压缩 + 自动压缩：
轮次 1-200：正常增长
自动触发：压缩为摘要
轮次 201-400：再次增长
自动触发：再次压缩
...
可以无限继续
```

---

## 7. 与 s05 的对比

| 方面 | s05 技能系统 | s06 上下文压缩 |
|------|-------------|----------------|
| 问题 | 静态知识浪费 Token | 动态对话消耗 Token |
| 解决方案 | 按需加载 | 三层压缩 |
| 触发时机 | LLM 请求时 | 每轮（微）+ 阈值（自动） |
| 存储位置 | skills/ 目录 | .transcripts/ 目录 |
| 目标 | 减少 Token 成本 | 实现无限会话 |

---

## 8. 常见问题

### Q1：为什么不在每轮都进行深度压缩？

**答案**：深度压缩的成本：
1. 需要 LLM 生成摘要（额外 API 调用）
2. 会丢失对话细节
3. 可能需要频繁恢复历史

微压缩的成本几乎为零，适合每轮执行。

### Q2：压缩后如何恢复之前的细节？

**答案**：
```python
# 从转录文件恢复
messages = load_transcript(TRANSCRIPT_DIR / "transcript_123456.jsonl")
# 然后继续使用这些消息
```

但通常不需要——摘要应该包含足够的信息来继续工作。

### Q3：Token 估算准确吗？

**答案**：不完全准确，但足够用于触发压缩：
1. 不同模型的 Token 算法不同
2. 估算通常偏大 10-20%
3. 宁可早压缩也不要晚压缩

### Q4：压缩会影响性能吗？

**答案**：
- 微压缩：不影响（只是字符串替换）
- 自动压缩：轻微影响（需要额外的 API 调用）
- 总体：压缩避免了上下文溢出，整体性能更好

---

## 9. 最佳实践

### 9.1 监控 Token 使用

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]

    while True:
        messages = micro_compact(messages)

        # 监控
        token_count = estimate_tokens(messages)
        if token_count % 10000 == 0:  # 每 10000 打印一次
            print(f"[Token 监控] 当前: {token_count}")

        if token_count > COMPACT_THRESHOLD:
            messages = auto_compact(messages)
        # ...
```

### 9.2 智能摘要

```python
def auto_compact(messages: list, context: dict) -> list:
    """带上下文的智能压缩"""
    summary_prompt = f"""生成会话摘要。

当前上下文：
- 用户目标：{context.get('goal')}
- 已完成任务：{context.get('completed')}
- 待办任务：{context.get('pending')}

请基于这些信息生成摘要...
"""
    # ...
```

### 9.3 分段压缩

```python
def segmented_compact(messages: list, segment_size: int = 20):
    """分段压缩，保留最近几段完整"""
    # 保留最近 20 轮完整
    recent = messages[-segment_size:]
    # 压缩更早的部分
    older = messages[:-segment_size]
    compressed = summarize(older)
    return compressed + recent
```

---

## 10. 小结

### 10.1 核心要点

| 要点 | 说明 |
|------|------|
| **微压缩** | 每轮执行，替换旧工具结果 |
| **自动压缩** | 阈值触发，生成摘要替换历史 |
| **手动压缩** | Agent 主动调用 |
| **转录存档** | 保存完整历史到磁盘 |

### 10.2 代码模板

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]

    while True:
        # 层 1：微压缩
        messages = micro_compact(messages)

        # 层 2：自动压缩
        if estimate_tokens(messages) > THRESHOLD:
            messages = auto_compact(messages)

        # 正常循环
        response = client.messages.create(messages=messages, ...)

        # 层 3：检查手动压缩
        if uses_compact_tool(response):
            messages = auto_compact(messages)
        # ...
```

### 10.3 关键洞察

```
上下文压缩的核心思想：
1. 不同年龄的信息价值不同
2. 旧工具结果可以替换为占位符
3. 整个对话可以浓缩为摘要
4. 完整历史可以存档到磁盘

设计原则：
- 轻量压缩每轮执行
- 重量压缩按需触发
- 存档保证可恢复性
- 摘要保持上下文连续性
```

---

## 11. 练习

### 练习 1：改进 Token 估算

实现更精确的 Token 估算，考虑不同内容类型的差异。

### 练习 2：压缩策略选择

添加压缩级别参数（light/medium/aggressive），调整压缩强度。

### 练习 3：语义压缩

使用嵌入模型对消息进行聚类，保留语义多样化的消息。

### 练习 4：压缩预览

实现 `compact_preview` 工具，让 Agent 在压缩前看到摘要，决定是否执行。

---

## 12. 第一阶段总结

恭喜！你完成了第一阶段的学习（s01-s06）：

| 章节 | 主题 | 掌握的技能 |
|------|------|------------|
| s01 | Agent 循环 | 最小可用 Agent |
| s02 | 工具系统 | 多工具分发器 |
| s03 | 任务规划 | TodoManager |
| s04 | 子 Agent | 上下文隔离 |
| s05 | 技能系统 | 按需加载知识 |
| s06 | 上下文压缩 | 无限会话 |

**你现在已经可以构建一个功能完整的 AI 编程助手！**

---

## 13. 下一步

第一阶段的 Agent 是单次会话的，任务状态只存在内存中。接下来我们将进入**持久化**阶段。

在 [s07：任务系统](./s07-task-system.md) 中，我们将学习：
- 如何持久化任务到磁盘
- 任务依赖关系（DAG）
- 跨会话的状态管理

---

< 上一章：[s05-技能系统](./s05-skill-loading.md) | [目录](./index.md) | 下一章：[s07-任务系统](./s07-task-system.md) >
