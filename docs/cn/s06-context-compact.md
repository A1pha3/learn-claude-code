# s06：上下文压缩 —— 实现无限会话

> **核心口号**：*"上下文会填满，需要战略性地遗忘"*

> **学习目标**：理解上下文窗口的限制，掌握三层压缩策略的精确实现，学会设计可持续的长对话机制

---

## 学习目标

完成本章后，你将能够：

1. **理解上下文饱和问题** —— 为什么长对话会"变笨"
2. **掌握三层压缩策略** —— 微压缩、自动压缩、手动压缩的精确实现
3. **理解 Token 估算机制** —— 字符数/4 的简化估算及其精度局限
4. **实现转录存档** —— JSONL 格式的完整历史保存与恢复
5. **掌握触发机制** —— KEEP_RECENT 和 THRESHOLD 参数的调优

### 学习分层指南

| 层级 | 掌握标准 | 对应内容 | 建议用时 |
|------|----------|----------|----------|
| L1 基础 | 能说明三层压缩分别解决哪类上下文问题，能运行示例并观察 Token 变化 | 1.问题的本质、2.微压缩、4.自动压缩、5.手动压缩 | 30 分钟 |
| L2 进阶 | 能独立实现 micro_compact 和 auto_compact，能根据场景调优 KEEP_RECENT 和 THRESHOLD | 3.Token 估算、6.转录存档、7.三层压缩的协同 | 45 分钟 |
| L3 专家 | 能设计分段压缩、智能摘要等进阶策略，能构建面向长会话的完整上下文管理体系 | 10.最佳实践、12.练习 | 60 分钟 |

---

## 0. 上手演练（建议先做）

先运行上下文压缩示例，再对照三层策略阅读：

```bash
uv run python agents/s06_context_compact.py
```

建议观察：

1. 微压缩在每轮结束后如何处理旧工具结果（观察 `[Previous: used {tool_name}]` 日志）。
2. 自动压缩在什么 Token 阈值触发，以及触发后的上下文变化。
3. `.transcripts/` 目录中的 JSONL 文件如何记录完整历史。

### 本章完成标准

- 能判断三层压缩分别解决哪一类上下文问题。
- 能根据会话长度给出合理的 KEEP_RECENT 和 THRESHOLD 参数。
- 能解释"压缩"和"可追溯"为什么必须同时成立。
- 能对照源码说明微压缩中 tool_name 映射的精确实现。

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

总计：~12,000 tokens（仅 5 轮）
```

**问题**：对话越长，上下文越大，最终可能超出窗口限制（200,000 tokens）。在复杂编程任务中，50-100 轮对话很常见，Token 消耗可能轻松超过 100,000。

### 1.2 饱和的后果

```
上下文窗口（200,000 tokens）：

┌─────────────────────────────────────┐
│ 系统提示（1,000 tokens）████        │
│ 用户问题（500 tokens）██            │
│ 工具调用 1（100 tokens）             │
│ 工具结果 1（4,000 tokens）██████████│  ← 这些旧结果已经没用了
│ 工具调用 2（100 tokens）             │     但仍然占据上下文空间
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
3. Token 预算耗尽 → 无法继续对话（API 报错）
4. API 费用线性增长 → 每次调用都发送全部历史

### 1.3 解决方案：三层压缩

```
Layer 1：微压缩 micro_compact（每轮执行，零成本）
  用占位符替换旧的工具结果
  "[Previous: used read_file]" ← 原来 4000 tokens 的文件内容

Layer 2：自动压缩 auto_compact（阈值触发）
  Token 估算超过 50000 时触发
  保存完整记录到 .transcripts/ 目录
  用 LLM 生成的摘要替换整个对话历史

Layer 3：手动压缩 compact 工具（Agent 主动调用）
  Agent 决定现在需要压缩
  调用 compact 工具 → 触发与自动压缩相同的逻辑
```

---

## 2. 微压缩（Micro-compact）—— Layer 1

### 2.1 核心思想

工具调用的结果（如文件内容、命令输出）在后续轮次中通常不再需要。微压缩在每轮 LLM 调用前，将旧的工具结果替换为简短的占位符。

### 2.2 源码实现分析

```python
KEEP_RECENT = 3  # 保留最近 3 个工具结果不压缩

def micro_compact(messages: list) -> list:
    # 步骤 1：收集所有 tool_result 的位置信息
    tool_results = []
    for msg_idx, msg in enumerate(messages):
        if msg["role"] == "user" and isinstance(msg.get("content"), list):
            for part_idx, part in enumerate(msg["content"]):
                if isinstance(part, dict) and part.get("type") == "tool_result":
                    tool_results.append((msg_idx, part_idx, part))

    # 步骤 2：如果工具结果不超过 KEEP_RECENT，不需要压缩
    if len(tool_results) <= KEEP_RECENT:
        return messages

    # 步骤 3：构建 tool_use_id → tool_name 的映射表
    #         遍历所有 assistant 消息中的 tool_use block
    tool_name_map = {}
    for msg in messages:
        if msg["role"] == "assistant":
            content = msg.get("content", [])
            if isinstance(content, list):
                for block in content:
                    if hasattr(block, "type") and block.type == "tool_use":
                        tool_name_map[block.id] = block.name

    # 步骤 4：压缩旧的（超出 KEEP_RECENT 的）工具结果
    to_clear = tool_results[:-KEEP_RECENT]  # 保留最后 3 个
    for _, _, result in to_clear:
        if isinstance(result.get("content"), str) and len(result["content"]) > 100:
            tool_id = result.get("tool_use_id", "")
            tool_name = tool_name_map.get(tool_id, "unknown")
            result["content"] = f"[Previous: used {tool_name}]"

    return messages
```

### 2.3 关键实现细节

**1. 双层索引定位**：`tool_results` 存储 `(msg_idx, part_idx, part)` 三元组。这是因为 `tool_result` 嵌套在 `user` 消息的 `content` 列表中（Anthropic API 规范：tool_result 必须作为 user 消息的一部分发送）。

**2. tool_name 映射**：微压缩不是简单地写 "tool result"，而是通过匹配 `tool_use_id` 找到对应的工具名，生成更有信息量的占位符：

```
[Previous: used read_file]     ← 比 [Previous: tool result] 更有用
[Previous: used bash]          ← LLM 能理解这是旧命令输出
[Previous: used edit_file]     ← 知道之前的编辑操作
```

**3. 长度阈值保护**：只有当原始内容长度 > 100 字符时才压缩。短结果（如 "Wrote 42 bytes"）本身就很短，压缩的收益不大。

**4. KEEP_RECENT = 3 的含义**：保留最近 3 个工具结果的完整内容。这个值平衡了上下文保留和压缩效果——最近的工具结果通常与当前任务最相关。

**5. 原地修改**：注意 `micro_compact` 直接修改 messages 列表中的元素（`result["content"] = ...`），不创建新列表。这是因为 `tool_results` 中的 `part` 是对原始消息对象的引用。

### 2.4 压缩效果计算

```
假设：
- 平均每个工具结果 2000 tokens
- 10 轮对话后积累了 20 个工具结果
- KEEP_RECENT = 3

压缩前：20 * 2000 = 40,000 tokens
压缩后：17 * ~5 + 3 * 2000 = 85 + 6000 = 6,085 tokens
节省：40,000 - 6,085 = 33,915 tokens (84.8%)
```

---

## 3. Token 估算

### 3.1 源码实现

```python
def estimate_tokens(messages: list) -> int:
    """Rough token count: ~4 chars per token."""
    return len(str(messages)) // 4
```

### 3.2 精度分析

这个估算方法非常简化——它将整个 messages 列表转为字符串，然后除以 4。其精度特点：

1. **~4 字符/token 是粗略近似**：不同模型的 tokenizer 差异很大。Claude 使用自己的 tokenizer，GPT-4 使用 cl100k_base，Llama 使用 sentencepiece。实际比例可能在 3-5 字符/token 之间浮动。
2. **`str(messages)` 的开销**：将 Python 对象转为字符串会包含 JSON 结构字符（花括号、引号、逗号等），这些不是真实的 token，导致估算偏高。
3. **`hasattr`/`block` 对象**：Anthropic SDK 返回的 content block 是自定义对象，`str()` 转换可能产生冗余文本。
4. **总体偏差**：估算通常偏高 10-30%，这意味着实际 Token 数可能低于估算值。

### 3.3 为什么不使用精确 tokenizer？

1. **避免额外依赖**：不需要安装 tiktoken 等库
2. **跨模型兼容**：不绑定特定模型的 tokenizer
3. **偏大比偏小安全**：提前触发压缩，避免实际超出窗口限制
4. **性能考虑**：`len(str()) // 4` 比 tokenizer 编码快几个数量级

### 3.4 改进方向

如果需要更精确的估算，可以考虑：

```python
# 方案 A：使用 Anthropic 的 Token 计数 API（最精确，但有网络开销）
response = client.count_tokens(model=MODEL, messages=messages)
token_count = response.input_tokens

# 方案 B：使用 tiktoken 近似（离线，较快）
import tiktoken
encoder = tiktoken.get_encoding("cl100k_base")
token_count = sum(len(encoder.encode(str(msg))) for msg in messages)
```

---

## 4. 自动压缩（Auto-compact）—— Layer 2

### 4.1 触发条件

```python
THRESHOLD = 50000  # Token 阈值

# 在 agent_loop 中，每轮调用前检查：
if estimate_tokens(messages) > THRESHOLD:
    print("[auto_compact triggered]")
    messages[:] = auto_compact(messages)
```

注意 `messages[:] =` 的写法——这是原地替换列表内容，确保外部引用（如 `history` 列表）也能看到更新。

### 4.2 源码实现分析

```python
TRANSCRIPT_DIR = WORKDIR / ".transcripts"

def auto_compact(messages: list) -> list:
    # 步骤 1：保存完整记录到磁盘
    TRANSCRIPT_DIR.mkdir(exist_ok=True)
    transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with open(transcript_path, "w") as f:
        for msg in messages:
            f.write(json.dumps(msg, default=str) + "\n")
    print(f"[transcript saved: {transcript_path}]")

    # 步骤 2：请求 LLM 生成摘要
    conversation_text = json.dumps(messages, default=str)[:80000]
    response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content":
            "Summarize this conversation for continuity. Include: "
            "1) What was accomplished, 2) Current state, "
            "3) Key decisions made. "
            "Be concise but preserve critical details.\n\n" + conversation_text}],
        max_tokens=2000,
    )
    summary = response.content[0].text

    # 步骤 3：用摘要替换整个消息历史
    return [
        {"role": "user", "content":
            f"[Conversation compressed. Transcript: {transcript_path}]\n\n{summary}"},
        {"role": "assistant", "content":
            "Understood. I have the context from the summary. Continuing."},
    ]
```

### 4.3 关键实现细节

**1. `json.dumps(msg, default=str)`**：使用 `default=str` 处理不可序列化的对象。Anthropic SDK 返回的 TextBlock、ToolUseBlock 等自定义对象不是原生可 JSON 序列化的，`default=str` 将它们转为字符串表示。

**2. `conversation_text[:80000]`**：截断到 80,000 字符。这是一个安全措施——如果对话非常长，完整的 JSON 可能超出 LLM 的输入窗口。80,000 字符约等于 20,000 tokens，在摘要请求的上下文中是合理的。

**3. 摘要提示词的结构**：明确要求 LLM 包含三个方面：
   - 已完成的工作（What was accomplished）
   - 当前状态（Current state）
   - 关键决策（Key decisions）

**4. 返回格式**：压缩后的消息列表只有 2 条消息：
   - `user` 消息：包含压缩标记、转录文件路径和摘要内容
   - `assistant` 消息：确认收到摘要，准备继续工作

**5. 额外 API 调用的成本**：自动压缩需要一次额外的 LLM 调用来生成摘要（消耗约 2000 output tokens），但这换来的是后续所有调用都使用精简的上下文，整体上节省大量 Token。

### 4.4 压缩效果

```
压缩前：50,000+ tokens（完整对话历史）
摘要请求：~20,000 input + ~1,000 output = ~21,000 tokens（一次性）
压缩后：~1,500 tokens（2 条消息：摘要 + 确认）

后续每次调用的节省：50,000 - 1,500 = 48,500 tokens
假设压缩后再进行 10 轮对话：总共节省 485,000 tokens
减去摘要成本 21,000 tokens → 净节省 464,000 tokens
```

---

## 5. 手动压缩（Manual-compact）—— Layer 3

### 5.1 工具定义

```python
{"name": "compact",
 "description": "Trigger manual conversation compression.",
 "input_schema": {
     "type": "object",
     "properties": {
         "focus": {
             "type": "string",
             "description": "What to preserve in the summary"
         }
     }
 }}
```

### 5.2 触发机制

手动压缩的触发流程与自动压缩不同，它发生在工具调用结果被添加到消息之后：

```python
# 在 agent_loop 中
results = []
manual_compact = False
for block in response.content:
    if block.type == "tool_use":
        if block.name == "compact":
            manual_compact = True
            output = "Compressing..."
        else:
            handler = TOOL_HANDLERS.get(block.name)
            output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
        results.append({"type": "tool_result", ...})

messages.append({"role": "user", "content": results})

# Layer 3：手动压缩触发点
if manual_compact:
    print("[manual compact]")
    messages[:] = auto_compact(messages)
```

### 5.3 关键实现细节

1. **compact 工具的处理器不执行实际压缩**：`TOOL_HANDLERS["compact"]` 只返回 `"Manual compression requested."` 字符串。实际压缩在 `agent_loop` 中通过检查 `manual_compact` 标志来触发。
2. **压缩在 tool_result 添加之后执行**：这确保了 compact 工具调用本身也被记录在转录文件中。
3. **复用 auto_compact 逻辑**：手动压缩调用的是同一个 `auto_compact()` 函数，共享转录保存和摘要生成逻辑。
4. **focus 参数**：虽然工具定义中有 `focus` 参数，但在当前实现中并未被使用。这是一个扩展点——可以在摘要提示词中加入 focus 参数，引导 LLM 保留特定方面的信息。

### 5.4 Agent 的使用场景

```
LLM：我已经完成了第一个阶段的开发，现在进入第二阶段。
      详细的文件读取记录不再需要了。
      tool_use: compact()

工具结果：Compressing...
[manual compact]
[transcript saved: .transcripts/transcript_1712000000.jsonl]

LLM：Understood. I have the context from the summary. Continuing.
     继续第二阶段...
```

**适用场景**：
- 完成一个大的任务阶段后，清理前阶段的详细信息
- Agent 识别到上下文过长，主动压缩以保持性能
- 在开始全新话题前，清除旧话题的痕迹

---

## 6. 转录存档

### 6.1 为什么要保存完整历史

压缩会丢失细节，但完整历史可能有价值：
- **调试**：查看 Agent 的完整决策过程
- **审计**：了解做了哪些文件修改
- **学习**：分析 Agent 的行为模式和效率
- **恢复**：在必要时回退到压缩前的状态

### 6.2 JSONL 存储格式

转录文件使用 JSONL（JSON Lines）格式，每行一个独立的 JSON 对象：

```jsonl
{"role": "user", "content": "修复登录 bug"}
{"role": "assistant", "content": [{"type": "text", "text": "Let me check the code..."}, {"type": "tool_use", "id": "toolu_01", "name": "read_file", "input": {"path": "auth.py"}}]}
{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_01", "content": "def authenticate(user, password):\n    ..."}]}
{"role": "assistant", "content": [{"type": "text", "text": "Found the issue..."}, {"type": "tool_use", "id": "toolu_02", "name": "edit_file", "input": {"path": "auth.py", "old_text": "...", "new_text": "..."}}]}
{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_02", "content": "Edited auth.py"}]}
...
```

**JSONL 格式的优势**：
1. **逐行可解析**：每行是完整的 JSON，不需要解析整个文件
2. **追加友好**：可以直接在文件末尾追加新消息
3. **容错性好**：某一行损坏不影响其他行的解析
4. **流式处理**：可以逐行读取处理大文件，不需要全部加载到内存
5. **兼容 `json.loads`**：每行直接 `json.loads(line)` 即可

### 6.3 文件命名规则

```python
transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
```

文件名使用 Unix 时间戳，例如：
- `transcript_1712000000.jsonl` — 第一次压缩时保存
- `transcript_1712001000.jsonl` — 第二次压缩时保存

**排序性**：时间戳保证文件名按时间顺序排列，方便查找最新的转录文件。

### 6.4 恢复机制

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

> **警告：恢复的消息列表不能直接传给 API**
>
> `load_transcript()` 恢复的消息中可能包含 Anthropic SDK 的自定义对象（如 TextBlock、ToolUseBlock）。这些对象在保存时被 `default=str` 转为了字符串表示，恢复后变成了普通字符串而非 SDK 对象。如果直接将恢复的消息列表传给 `client.messages.create()`，API 会因为无法识别这些对象的格式而报错。
>
> 因此，恢复的转录文件仅适用于以下场景：
> - 人工审查和调试（打印、搜索、分析）
> - 提取关键信息后手动构建新的消息列表
> - 构建专门的适配层将字符串转回 SDK 对象（高级用法）

**实际用途**：
- 人工审查 Agent 的完整决策过程
- 分析哪些操作是成功的、哪些是失败的
- 提取关键信息（如修改了哪些文件、执行了哪些命令）

---

## 7. 三层压缩的协同

### 7.1 在 agent_loop 中的执行顺序

```python
def agent_loop(messages: list):
    while True:
        # Layer 1：微压缩（每轮都执行）
        micro_compact(messages)

        # Layer 2：自动压缩（Token 阈值触发）
        if estimate_tokens(messages) > THRESHOLD:
            print("[auto_compact triggered]")
            messages[:] = auto_compact(messages)

        # 正常 LLM 调用
        response = client.messages.create(...)
        messages.append({"role": "assistant", "content": response.content})

        # 检查是否需要继续循环
        if response.stop_reason != "tool_use":
            return

        # 处理工具调用
        results = []
        manual_compact = False
        for block in response.content:
            if block.type == "tool_use":
                if block.name == "compact":
                    manual_compact = True
                    output = "Compressing..."
                else:
                    output = handler(**block.input)
                results.append({"type": "tool_result", ...})
        messages.append({"role": "user", "content": results})

        # Layer 3：手动压缩（Agent 主动调用）
        if manual_compact:
            messages[:] = auto_compact(messages)
```

### 7.2 执行流程图

```
每轮循环：
┌─────────────────────────────────────┐
│ Layer 1：微压缩 micro_compact       │
│   - 收集所有 tool_result 位置       │
│   - 构建 tool_use_id → name 映射    │
│   - 替换超过 KEEP_RECENT 的旧结果   │
│   - 成本：~0（纯字符串替换）        │
└─────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ Token 预算检查                      │
│   - estimate_tokens(messages)       │
│   - 返回值 > THRESHOLD (50000)？    │
└─────────────────────────────────────┘
           │
           ├── 否 ──→ 继续正常循环
           │
           └── 是 ──▼
┌─────────────────────────────────────┐
│ Layer 2：自动压缩 auto_compact      │
│   - 保存完整记录到 .transcripts/    │
│   - 请求 LLM 生成摘要              │
│   - 用 [摘要 + 确认] 替换全部历史   │
│   - 成本：1 次 LLM 调用            │
└─────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ 正常 LLM 调用                      │
│   - 发送 messages + tools + system  │
│   - 接收响应                       │
└─────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ 工具调用处理                        │
│   - 若调用 compact → 设置标志      │
│   - 其他工具 → 正常执行            │
└─────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ Layer 3：手动压缩检查               │
│   - compact 标志为 True？           │
│   - 是 → 执行 auto_compact         │
└─────────────────────────────────────┘
```

### 7.3 参数调优

| 参数 | 源码值 | 说明 | 调优建议 |
|------|--------|------|----------|
| `KEEP_RECENT` | 3 | 微压缩保留的最近工具结果数 | 复杂任务可增至 5-10；简单任务可降至 2 |
| `THRESHOLD` | 50000 | 自动压缩的 Token 阈值 | 取决于模型窗口大小，建议设为窗口的 25%-40% |
| `max_tokens`（摘要） | 2000 | 摘要的最大 Token 数 | 复杂任务可增至 4000；简单场景可降至 1000 |
| 截断长度 | 80000 字符 | 摘要请求中对话文本的最大长度 | 约等于 20000 tokens，可根据模型窗口调整 |

### 7.4 效果对比

```
无压缩：
  轮次 1-50：正常增长
  轮次 51：超出窗口，API 报错，无法继续

有微压缩（Layer 1 only）：
  轮次 1-100：缓慢增长（旧结果被替换为占位符）
  轮次 101：仍然超出窗口（消息数量本身也在增长）

有微压缩 + 自动压缩（Layer 1 + 2）：
  轮次 1-200：正常增长
  自动触发：50,000 tokens → 压缩为 ~1,500 tokens
  轮次 201-400：再次增长
  自动触发：再次压缩
  ...
  可以无限继续

有微压缩 + 自动压缩 + 手动压缩（Layer 1 + 2 + 3）：
  同上 + Agent 可以在阶段切换时主动清理
  更精确地控制上下文内容
```

---

## 8. 与 s05 的对比

| 方面 | s05 技能系统 | s06 上下文压缩 |
|------|-------------|----------------|
| 问题 | 静态知识浪费 Token | 动态对话消耗 Token |
| 解决方案 | 两层注入（按需加载） | 三层压缩（按需遗忘） |
| 触发时机 | LLM 请求时（load_skill） | 每轮（微）+ 阈值（自动）+ 主动（手动） |
| 存储位置 | skills/ 目录 | .transcripts/ 目录 |
| 文件格式 | SKILL.md（Markdown） | transcript_*.jsonl（JSON Lines） |
| 目标 | 减少 Token 成本 | 实现无限会话 |
| 互补性 | 控制知识注入量 | 控制上下文膨胀量 |

**协同设计**：s05 控制的是"知识进入上下文的方式"（按需加载），s06 控制的是"上下文随时间增长的方式"（按需遗忘）。两者结合，使得 Agent 既能获取需要的知识，又不会因历史积累而崩溃。

---

## 9. 常见问题

### Q1：为什么不在每轮都进行深度压缩？

**答案**：深度压缩的成本太高：
1. 每次压缩需要一次额外的 LLM API 调用（~21,000 tokens）
2. 会丢失对话细节（摘要可能遗漏重要信息）
3. 如果每轮都压缩，API 调用次数翻倍

微压缩的成本几乎为零（只是字符串替换），适合每轮执行。深度压缩只在真正需要时触发。

### Q2：压缩后如何恢复之前的细节？

**答案**：
```python
# 从转录文件恢复（用于人工审查，不能直接传给 API）
messages = load_transcript(Path(".transcripts/transcript_1712000000.jsonl"))
```

但通常不需要恢复——摘要应该包含足够的信息来继续工作。转录文件主要用于事后分析和调试。

### Q3：Token 估算准确吗？

**答案**：不完全准确，但对于触发压缩来说够用了：
1. `len(str(messages)) // 4` 是粗略近似，实际比例因模型而异
2. 估算通常偏高 10-30%（包含 JSON 结构字符）
3. 偏大比偏小安全——宁可早压缩也不要晚压缩
4. 如果需要精确值，应使用 Anthropic 的 Token 计数 API

### Q4：压缩会影响 Agent 的性能吗？

**答案**：
- **微压缩**：几乎无影响。只是字符串替换，不改变消息结构。占位符 `[Previous: used read_file]` 保留了足够的语义信息。
- **自动压缩**：有一次性成本（1 次 LLM 调用），但换来后续所有调用的上下文缩减。
- **总体**：压缩避免了上下文溢出和性能退化，整体上让 Agent 保持更好的表现。

### Q5：KEEP_RECENT = 3 够用吗？

**答案**：取决于任务复杂度：
- **简单任务**（3-5 轮就能完成）：3 个足够
- **中等任务**（10-20 轮）：5-8 个更合适，避免丢失关键上下文
- **复杂任务**（50+ 轮）：建议 10 个，同时配合更低的 THRESHOLD 触发自动压缩

### Q6：压缩后 Agent 行为异常怎么办？

压缩（尤其是自动压缩）会丢失大量对话细节，可能导致 Agent 出现以下问题：

| 异常表现 | 根因 | 解决方法 |
|----------|------|----------|
| Agent 忘记之前的工作 | 摘要遗漏了已完成任务的细节 | 在摘要提示词中强调"列出所有已完成的工作"；或降低 THRESHOLD 让压缩更早触发（减少单次压缩的信息损失） |
| 重复已完成任务 | 摘要未记录已完成的步骤 | 手动调用 compact 时使用 focus 参数，明确指定保留哪些进度信息 |
| 偏离原始目标 | 摘要丢失了用户的初始需求 | 摘要提示词中增加"完整保留用户的原始请求"；或在 system prompt 中固定关键目标 |

**通用排查步骤**：

1. 查看 `.transcripts/` 中最新的转录文件，确认压缩前的完整上下文
2. 检查摘要内容是否遗漏了关键信息（任务进度、用户需求、重要决策）
3. 如果频繁出现异常，考虑调整参数：增大 KEEP_RECENT、降低 THRESHOLD、或在摘要提示词中增加更多结构化要求
4. 极端情况下，可以从转录文件恢复完整上下文，重新构建消息列表

---

## 10. 最佳实践

### 10.1 监控 Token 使用

```python
# 在 agent_loop 中添加日志
def agent_loop(messages: list):
    while True:
        messages = micro_compact(messages)
        token_count = estimate_tokens(messages)
        print(f"[Token 估算: {token_count}]")  # 添加监控日志

        if token_count > THRESHOLD:
            messages[:] = auto_compact(messages)
        # ...
```

### 10.2 智能摘要（改进方向）

当前实现使用通用摘要提示词。可以加入上下文感知：

```python
def auto_compact(messages: list, focus: str = None) -> list:
    prompt = "Summarize this conversation for continuity."
    if focus:
        prompt += f" Pay special attention to: {focus}"
    # ...
```

这样手动压缩时可以通过 `focus` 参数引导摘要保留特定信息。

### 10.3 分段压缩（改进方向）

当前实现是"全量压缩"——将所有消息替换为摘要。更精细的方式是分段压缩：

```python
def segmented_compact(messages: list, keep_recent: int = 20):
    """保留最近 N 条消息完整，只压缩更早的部分"""
    recent = messages[-keep_recent:]
    older = messages[:-keep_recent]
    compressed_summary = summarize(older)
    return [
        {"role": "user", "content": f"[Earlier summary]\n{compressed_summary}"},
        {"role": "assistant", "content": "Understood."},
    ] + recent
```

---

## 实战应用：让 Agent 支持长时间会话

前面的章节从原理层面解释了三层压缩的设计和实现。下面通过三个真实场景，帮助你为不同类型的长时间会话选择正确的压缩策略。

### 场景 1：全天候编程助手

```
场景描述：开发者使用 Agent 作为日常编程助手，8 小时工作中持续对话

典型的对话模式：
  上午 9:00  - 修复一个 bug（5 轮，~15,000 tokens）
  上午 10:30 - 完成一个小功能（8 轮，~20,000 tokens）
  下午 1:00  - 重构代码（15 轮，~40,000 tokens）
  下午 3:00  - 写测试（10 轮，~25,000 tokens）
  下午 5:00  - 代码审查（6 轮，~12,000 tokens）

无压缩时：累计 ~112,000 tokens，上下文接近饱和
  后果：Agent 开始忘记上午的决策，系统提示被淹没

有微压缩 + 自动压缩时：
  每个任务结束后微压缩已清理旧工具结果
  在累计 ~50,000 tokens 时自动压缩触发
  压缩后上下文降至 ~1,500 tokens
  全天可能触发 2-3 次自动压缩
  每次压缩后 Agent 能继续正常工作

关键配置：KEEP_RECENT=3, THRESHOLD=50000
  适合"任务切换频繁，每个任务独立"的工作模式
```

### 场景 2：大规模代码重构

```
场景描述：对 100+ 个文件进行渐进式重构（统一命名风格）

对话特点：
  - 需要保持全局视野：知道哪些文件已改、哪些未改
  - 单次操作简单：每次改几个文件的命名
  - 但操作次数极多：可能需要 200+ 轮对话

压缩挑战：
  - 微压缩会清理旧的 edit_file 结果（正合需要）
  - 但自动压缩的摘要可能遗漏"哪些文件已修改"的进度信息
  - 如果摘要不完整，Agent 可能重复修改已完成的文件

推荐策略：
  1. KEEP_RECENT=5 -- 多保留一些最近的编辑上下文
  2. THRESHOLD=40000 -- 更早触发压缩，减少单次压缩的信息损失
  3. 在 system prompt 中维护一个"已完成文件列表"，
     压缩不会影响 system prompt
  4. 鼓励 Agent 在阶段切换时主动调用 compact()，
     并在 focus 参数中指定保留进度信息

关键配置：KEEP_RECENT=5, THRESHOLD=40000, 使用 manual compact
```

### 场景 3：故障排查会话

```
场景描述：生产环境故障排查，需要多轮调查、保留关键发现、丢弃中间细节

对话模式：
  轮次 1-5：  读取日志文件（产生大量工具结果）
  轮次 6-10： 分析错误模式（需要前面几轮的上下文）
  轮次 11-15：定位根因（前面读的日志不再需要）
  轮次 16-20：实施修复（需要根因分析结论，不需要原始日志）
  轮次 21-25：验证修复（需要修复方案，不需要根因分析过程）

压缩策略的核心需求：
  - 保留关键发现（根因、修复方案）
  - 丢弃中间细节（日志内容、中间假设）
  - 自动压缩的摘要必须包含"关键发现"

推荐策略：
  1. KEEP_RECENT=3 -- 排查中每步只关心最近几轮的上下文
  2. THRESHOLD=60000 -- 允许更多上下文积累，
     因为排查需要前后关联的信息
  3. 在找到根因后主动调用 compact(focus="root cause and fix plan")
  4. 这样压缩后的摘要会重点保留根因和修复方案，
     而非排查过程中的大量日志内容

关键配置：KEEP_RECENT=3, THRESHOLD=60000, 找到根因后手动 compact
```

### 参数调优决策矩阵

根据会话类型选择不同的参数组合：

| 会话类型 | KEEP_RECENT | THRESHOLD | 推荐压缩策略 | 原因 |
|----------|-------------|-----------|--------------|------|
| 日常编程（频繁切换任务） | 3 | 50000 | 微压缩 + 自动压缩 | 任务独立，旧工具结果可安全丢弃 |
| 大规模重构（需保持进度） | 5-8 | 40000 | 微压缩 + 自动压缩 + 手动压缩 | 需要多保留最近上下文，主动压缩保留进度 |
| 故障排查（前后关联） | 3 | 60000 | 微压缩 + 手动压缩（找到关键发现后） | 允许更多上下文积累，关键节点手动压缩 |
| 数据分析（大量查询结果） | 2 | 30000 | 微压缩 + 频繁自动压缩 | 查询结果体量大，需要更积极地压缩 |
| 文档编写（长文本输出） | 3 | 50000 | 微压缩 + 自动压缩 | 工具调用较少，token 增长较慢 |
| 教学辅导（需要完整上下文） | 10 | 70000 | 微压缩 + 谨慎自动压缩 | 需要保留尽量多的对话历史 |

**调优原则**：

1. **KEEP_RECENT 越大**，保留的最近上下文越多，适合需要前后关联的任务（如故障排查、教学）。代价是微压缩效果减弱。
2. **THRESHOLD 越低**，压缩触发越频繁，适合工具输出体量大的场景（如数据分析）。代价是压缩摘要可能丢失细节。
3. **手动压缩是"精度工具"**：当自动压缩无法保证关键信息不丢失时，让 Agent 在关键节点主动压缩，并通过 focus 参数引导摘要方向。
4. **先保守后调整**：初始使用默认参数（KEEP_RECENT=3, THRESHOLD=50000），观察 Agent 行为后再针对性调整。

---

## 11. 小结

### 11.1 核心要点

| 要点 | 说明 |
|------|------|
| **Layer 1 微压缩** | 每轮执行，将超过 KEEP_RECENT (3) 的旧工具结果替换为占位符 |
| **Layer 2 自动压缩** | 当 estimate_tokens > THRESHOLD (50000) 时触发，保存转录、生成摘要 |
| **Layer 3 手动压缩** | Agent 主动调用 compact 工具，复用 auto_compact 逻辑 |
| **Token 估算** | `len(str(messages)) // 4`，粗略但安全（偏高 10-30%） |
| **转录存档** | JSONL 格式，每行一条消息，使用时间戳命名 |
| **tool_name 映射** | 微压缩通过匹配 tool_use_id 找到工具名，生成信息量更高的占位符 |

### 11.2 代码模板

```python
# 常量定义
KEEP_RECENT = 3          # 微压缩保留的最近工具结果数
THRESHOLD = 50000        # 自动压缩的 Token 阈值
TRANSCRIPT_DIR = Path(".transcripts")

def agent_loop(messages: list):
    while True:
        # Layer 1：微压缩（每轮）
        micro_compact(messages)

        # Layer 2：自动压缩（阈值触发）
        if estimate_tokens(messages) > THRESHOLD:
            messages[:] = auto_compact(messages)

        # 正常 LLM 调用
        response = client.messages.create(...)

        # Layer 3：手动压缩（Agent 主动调用 compact 工具）
        if manual_compact:
            messages[:] = auto_compact(messages)
```

### 11.3 关键洞察

```
上下文压缩的核心思想：
1. 不同年龄的信息价值不同——旧的工具结果可以安全丢弃
2. 微压缩是"外科手术"——精准替换单个工具结果
3. 自动压缩是"核弹"——用摘要替换整个历史
4. 手动压缩是"主动防御"——Agent 自主决定何时清理
5. 转录存档保证"可恢复性"——压缩不等于永久丢失

设计原则：
- 轻量压缩每轮执行（微压缩，零成本）
- 重量压缩按需触发（自动压缩，有成本但值得）
- 存档保证可追溯（JSONL 转录文件）
- 摘要保持上下文连续性（LLM 生成的结构化摘要）
- 宁可早压缩也不要晚压缩（估算偏高是安全的）
```

---

## 12. 练习

### 练习 1：改进 Token 估算

实现更精确的 Token 估算：
1. 使用 Anthropic 的 Token 计数 API
2. 或者使用 `len(str(messages)) // 3.5` 等更准确的比例
3. 对比两种方法的精度差异

### 练习 2：压缩策略选择

添加压缩级别参数（light/medium/aggressive），调整：
- light：KEEP_RECENT=5, THRESHOLD=80000
- medium：KEEP_RECENT=3, THRESHOLD=50000
- aggressive：KEEP_RECENT=1, THRESHOLD=30000

### 练习 3：利用 focus 参数

在手动压缩时利用 compact 工具的 `focus` 参数，引导摘要保留特定方面的信息。

### 练习 4：压缩预览

实现 `compact_preview` 工具，让 Agent 在压缩前先看到摘要内容，再决定是否执行压缩。

---

## 13. 第一阶段总结

恭喜！你完成了第一阶段的学习（s01-s06）：

| 章节 | 主题 | 核心能力 |
|------|------|----------|
| s01 | Agent 循环 | 最小可用 Agent（while True 循环 + tool_use 分发） |
| s02 | 工具系统 | 多工具分发器（TOOL_HANDLERS 字典） |
| s03 | 任务规划 | TodoManager（内存列表 + 渲染文本 + 提醒机制） |
| s04 | 子 Agent | 上下文隔离（独立消息列表 + 结果摘要） |
| s05 | 技能系统 | 按需加载知识（两层注入 + SKILL.md） |
| s06 | 上下文压缩 | 无限会话（三层压缩 + JSONL 转录） |

**你现在已经可以构建一个功能完整的 AI 编程助手！** 它具备：
- 工具调用能力（读写文件、执行命令）
- 任务规划和跟踪能力
- 子任务委托和上下文隔离
- 按需加载专业知识
- 长对话的上下文管理

---

## 14. 下一步

第一阶段的 Agent 是单次会话的，任务状态只存在内存中。接下来我们将进入**持久化**阶段。

在 [s07：任务系统](./s07-task-system.md) 中，我们将学习：
- 如何持久化任务到磁盘
- 任务依赖关系（DAG）
- 跨会话的状态管理

---

< 上一章：[s05-技能系统](./s05-skill-loading.md) | [目录](./index.md) | 下一章：[s07-任务系统](./s07-task-system.md) >
