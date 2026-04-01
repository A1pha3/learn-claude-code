# s05：技能系统 —— 按需加载知识

> **核心口号**：*"不要把所有知识塞进系统提示，需要时再加载"*

> **学习目标**：理解两层注入模式，掌握 SkillLoader 的实现机制，学会高效的 Token 管理

---

## 学习目标

完成本章后，你将能够：

1. **理解两层注入模式** —— 元数据（Layer 1）与完整内容（Layer 2）的职责划分
2. **掌握 SkillLoader 实现** —— YAML frontmatter 解析、技能自动发现与注册
3. **实现按需加载** —— 通过 load_skill 工具在需要时注入详细知识
4. **量化 Token 收益** —— 计算两层注入相比全量注入的 Token 节省

---

## 0. 上手演练（建议先做）

先跑技能加载示例，再看两层注入实现：

```bash
uv run python agents/s05_skill_loading.py
```

建议观察：

1. 首轮系统提示是否只包含轻量技能目录（Layer 1）。
2. 何时触发 `load_skill`，以及加载后上下文如何变化。
3. 只按需注入技能内容时，Token 消耗如何下降。

### 本章完成标准

- 能清楚区分 Layer 1 元数据与 Layer 2 完整内容的职责。
- 能独立新增一个 SKILL.md 并让系统自动发现。
- 能说明"按需加载"在上下文成本与响应质量上的收益。
- 能对照源码解释 SkillLoader 的 frontmatter 解析逻辑。

---

## 1. 问题的本质

### 1.1 知识注入的困境

假设你希望 Agent 遵循多种专业规范：

```
需要的专业知识（完整内容）：
- code-review:  代码审查规范（~2000 tokens）
- pdf:          PDF 处理指南（~1500 tokens）
- agent-builder: Agent 构建方法论（~1800 tokens）
- mcp-builder:  MCP 服务器构建（~2000 tokens）
总计：~7300 tokens
```

**问题**：如果全部放入系统提示：

```python
SYSTEM = """你是一个编程助手。

代码审查规范：
[2000 tokens 的详细说明]

PDF 处理指南：
[1500 tokens 的详细说明]

Agent 构建方法论：
[1800 tokens 的详细说明]

...
"""
```

**后果**：
1. 每次调用都发送 7300+ tokens 的静态知识
2. 大部分对话只用到一个技能（其余全部浪费）
3. 上下文窗口被静态内容占据，留给动态对话的空间减少
4. 技能数量越多，浪费越严重——Token 成本线性增长

### 1.2 理想的方式

```
第一次调用（轻量）：
系统提示（~80 tokens）：
  你是一个编程助手。
  可用技能：
    - code-review: Perform thorough code reviews...
    - pdf: Process PDF files...
    - agent-builder: Design and build AI agents...

用户：帮我做一个 code review
  ↓
  LLM 决定加载 code-review 技能

第二次调用（按需）：
系统提示（~80 tokens）+ 工具结果（~2000 tokens）：
  <skill name="code-review">
  # Code Review Skill
  ...（完整的审查规范、检查清单、代码示例）
  </skill>

只加载需要的知识！
```

---

## 2. 两层注入模式

### 2.1 架构对比

```
❌ 单层注入（浪费）：
┌─────────────────────────────────────┐
│         系统提示（每次调用）         │
│                                     │
│  code-review: 完整内容（~2000t）    │ ← 大部分时间不需要
│  pdf: 完整内容（~1500t）           │
│  agent-builder: 完整内容（~1800t） │
│  mcp-builder: 完整内容（~2000t）   │
│  ...                                │
└─────────────────────────────────────┘
每次调用：~7300 tokens 固定成本

✅ 两层注入（高效）：
┌─────────────────────────────────────┐
│    系统提示（每次调用，轻量）        │
│                                     │
│  可用技能：                          │
│    - code-review: Perform...        │ ← 每个技能 ~20 tokens
│    - pdf: Process PDF files...      │    只占 ~80 tokens 总计
│    - agent-builder: Design...       │
│    - mcp-builder: Build MCP...      │
│                                     │
│  当需要使用某个技能时，调用          │
│  load_skill 工具加载完整内容        │
└─────────────────────────────────────┘
第一次调用：~80 tokens（技能目录）
需要时加载：+1500-2000 tokens（仅加载 1 个技能）
```

### 2.2 两层的内容设计

| 层级 | 内容 | Token 估算 | 何时加载 | 加载方式 |
|------|------|------------|----------|----------|
| **Layer 1（元数据）** | 技能名称 + 简短描述 + 可选标签 | ~20-30/skill | 每次 LLM 调用 | 写入系统提示 |
| **Layer 2（完整内容）** | 详细的规范、步骤、代码示例 | ~1500-2000/skill | LLM 主动请求时 | 通过 tool_result 返回 |

### 2.3 Token 节省计算

假设有 N 个技能，平均每个技能完整内容为 T_body tokens，元数据为 T_meta tokens：

```
单层注入（全量）：    Token/次 = N * T_body
                     4 个技能 = 4 * 1800 = 7200 tokens/次

两层注入（按需）：    Token/次 = N * T_meta + k * T_body（k = 本轮加载的技能数，通常 k=1）
                     4 个技能，加载 1 个 = 4 * 20 + 1 * 1800 = 1880 tokens/次

节省率 = (7200 - 1880) / 7200 = 73.9%

当技能数 N 增大时，节省率趋近于 100%（因为 N * T_body >> N * T_meta + T_body）
```

**关键洞察**：两层注入将 Token 成本从 O(N * T_body) 降低到 O(N * T_meta + k * T_body)，其中 k 远小于 N。

---

## 3. 技能文件结构与实际布局

### 3.1 技能目录的实际结构

对照项目中的 `skills/` 目录：

```
skills/
├── agent-builder/
│   ├── SKILL.md                    # 技能定义（frontmatter + 正文）
│   ├── references/                 # 可选：参考资料
│   │   ├── agent-philosophy.md     # Agent 哲学深入
│   │   ├── minimal-agent.py        # 最小 Agent 模板
│   │   ├── subagent-pattern.py     # 子 Agent 模式模板
│   │   └── tool-templates.py       # 工具定义模板
│   └── scripts/                    # 可选：辅助脚本
│       └── init_agent.py           # Agent 项目脚手架
├── code-review/
│   └── SKILL.md                    # 简单技能（无辅助资源）
├── mcp-builder/
│   └── SKILL.md
└── pdf/
    └── SKILL.md
```

**要点**：
1. 每个技能至少包含一个 `SKILL.md` 文件，位于以技能名命名的子目录中
2. `references/` 和 `scripts/` 是可选的辅助资源，不会被自动加载到上下文
3. SkillLoader 使用 `rglob("SKILL.md")` 递归扫描，因此支持任意深度的目录嵌套
4. 技能 ID 由 SKILL.md 的 frontmatter 中的 `name` 字段决定，若缺失则使用目录名

### 3.2 SKILL.md 的实际格式

以 `skills/code-review/SKILL.md` 为例：

```markdown
---
name: code-review
description: Perform thorough code reviews with security, performance, and maintainability analysis. Use when user asks to review code, check for bugs, or audit a codebase.
---

# Code Review Skill

You now have expertise in conducting comprehensive code reviews.

## Review Checklist

### 1. Security (Critical)
- [ ] Injection vulnerabilities
- [ ] Authentication issues
...

### 2. Correctness
- [ ] Logic errors
...
```

以 `skills/pdf/SKILL.md` 为例：

```markdown
---
name: pdf
description: Process PDF files - extract text, create PDFs, merge documents. Use when user asks to read PDF, create PDF, or work with PDF files.
---

# PDF Processing Skill

You now have expertise in PDF manipulation.

## Reading PDFs
...
```

**结构规范**：
1. **YAML frontmatter**（`---` 之间的部分）：包含 `name` 和 `description` 两个必需字段
2. **Markdown 正文**（`---` 之后的部分）：详细的技能内容，包含步骤、示例、最佳实践
3. `description` 应当写明触发条件（"Use when..."），帮助 LLM 判断何时需要加载该技能

### 3.3 YAML frontmatter 的可选字段

当前 SkillLoader 会解析所有 `key: value` 形式的字段到 meta 字典中：

```yaml
---
name: agent-builder                              # 必需：技能 ID
description: |                                   # 必需：技能描述
  Design and build AI agents...
  Keywords: agent, assistant, autonomous...
tags: agent, build, architecture                 # 可选：标签
---
```

`get_descriptions()` 方法会读取 `tags` 字段，若有值则追加在描述后面：

```python
# 源码 s05_skill_loading.py 第 91-95 行
line = f"  - {name}: {desc}"
if tags:
    line += f" [{tags}]"
```

因此 Layer 1 的输出可能形如：

```
可用技能：
  - agent-builder: Design and build AI agents... [agent, build, architecture]
  - code-review: Perform thorough code reviews...
  - pdf: Process PDF files...
```

---

## 4. SkillLoader 源码分析

### 4.1 类结构

```python
class SkillLoader:
    def __init__(self, skills_dir: Path):
        self.skills_dir = skills_dir
        self.skills = {}           # {name: {"meta": dict, "body": str, "path": str}}
        self._load_all()
```

核心数据结构是 `self.skills` 字典，键为技能名，值为包含 `meta`（元数据字典）、`body`（正文文本）和 `path`（文件路径）的字典。

### 4.2 自动发现机制

```python
def _load_all(self):
    if not self.skills_dir.exists():
        return
    for f in sorted(self.skills_dir.rglob("SKILL.md")):
        text = f.read_text()
        meta, body = self._parse_frontmatter(text)
        name = meta.get("name", f.parent.name)
        self.skills[name] = {"meta": meta, "body": body, "path": str(f)}
```

**关键行为**：
1. `rglob("SKILL.md")`：递归扫描 skills_dir 下所有名为 `SKILL.md` 的文件
2. `sorted()`：按文件路径排序，保证加载顺序的确定性
3. 若 skills_dir 不存在，直接返回（不报错），保证启动安全
4. `name` 优先取 frontmatter 中的值，若缺失则使用父目录名（`f.parent.name`）

### 4.3 Frontmatter 解析的实际实现

```python
def _parse_frontmatter(self, text: str) -> tuple:
    """Parse YAML frontmatter between --- delimiters."""
    match = re.match(r"^---\n(.*?)\n---\n(.*)", text, re.DOTALL)
    if not match:
        return {}, text
    meta = {}
    for line in match.group(1).strip().splitlines():
        if ":" in line:
            key, val = line.split(":", 1)
            meta[key.strip()] = val.strip()
    return meta, match.group(2).strip()
```

**注意：这不是真正的 YAML 解析器！** 这里使用的是简单的逐行 `split(":", 1)` 解析。其特点：

1. **不使用 yaml 库**：没有引入 `import yaml`，而是用正则 + 字符串分割实现轻量解析
2. **支持单行 key: value**：每行只解析第一个冒号，冒号后的所有内容作为值
3. **不支持嵌套结构**：不支持 YAML 的列表、字典嵌套
4. **支持多行值的管道语法**（`|`）：由于 `split(":", 1)` 的存在，`description: |` 后面的内容被截取为 `|`，而多行文本实际不会被正确解析——但 `agent-builder` 的 description 使用了 `|` 管道语法后跟多行文本，这种情况下只有第一行 `|` 被当作值
5. **正则 `^---\n(.*?)\n---\n(.*)`**：匹配以 `---\n` 开头、`\n---\n` 分隔、后面是正文的模式，使用 `re.DOTALL` 让 `.` 匹配换行符

**实际影响**：对于 `agent-builder` 的 SKILL.md，其 description 使用了 `|` 多行语法：

```yaml
description: |
  Design and build AI agents for any domain. Use when users:
  (1) ask to "create an agent"...
```

经过 `split(":", 1)` 解析后，`description` 的值只会是 `|`（管道符号本身），而不是完整的多行文本。这意味着在 Layer 1 的技能目录中，agent-builder 的描述可能只显示 `|` 而非完整描述。这是一个已知的简化实现的局限性。

**改进方向**（供扩展参考）：引入真正的 YAML 解析器可以正确处理多行值、列表、嵌套字典等复杂结构。

### 4.4 Layer 1 输出：get_descriptions()

```python
def get_descriptions(self) -> str:
    """Layer 1: short descriptions for the system prompt."""
    if not self.skills:
        return "(no skills available)"
    lines = []
    for name, skill in self.skills.items():
        desc = skill["meta"].get("description", "No description")
        tags = skill["meta"].get("tags", "")
        line = f"  - {name}: {desc}"
        if tags:
            line += f" [{tags}]"
        lines.append(line)
    return "\n".join(lines)
```

**输出示例**（对照实际 skills 目录）：

```
可用技能：
  - agent-builder: |
  - code-review: Perform thorough code reviews with security, performance, and maintainability analysis. Use when user asks to review code, check for bugs, or audit a codebase.
  - mcp-builder: Build MCP (Model Context Protocol) servers that give Claude new capabilities. Use when user wants to create an MCP server, add tools to Claude, or integrate external services.
  - pdf: Process PDF files - extract text, create PDFs, merge documents. Use when user asks to read PDF, create PDF, or work with PDF files.
```

### 4.5 Layer 2 输出：get_content()

```python
def get_content(self, name: str) -> str:
    """Layer 2: full skill body returned in tool_result."""
    skill = self.skills.get(name)
    if not skill:
        return f"Error: Unknown skill '{name}'. Available: {', '.join(self.skills.keys())}"
    return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

**关键行为**：
1. 用 `<skill name="...">...</skill>` XML 标签包裹正文，方便 LLM 识别内容边界
2. 错误时返回可用技能列表，帮助 LLM 纠正技能名
3. 只返回 `body`（frontmatter 之后的内容），不包含元数据

---

## 5. 系统提示的组装

```python
SKILL_LOADER = SkillLoader(SKILLS_DIR)

SYSTEM = f"""You are a coding agent at {WORKDIR}.
Use load_skill to access specialized knowledge before tackling unfamiliar topics.

Skills available:
{SKILL_LOADER.get_descriptions()}"""
```

**组装过程**：
1. `SkillLoader` 在初始化时完成所有技能的扫描和解析
2. `get_descriptions()` 的结果被嵌入系统提示字符串
3. 系统提示在每次 LLM 调用时都发送，因此 Layer 1 是"常驻"的
4. 系统提示中明确指示 LLM "在处理不熟悉的主题前先使用 load_skill"

---

## 6. load_skill 工具定义与注册

### 6.1 工具定义

```python
{"name": "load_skill",
 "description": "Load specialized knowledge by name.",
 "input_schema": {
     "type": "object",
     "properties": {
         "name": {
             "type": "string",
             "description": "Skill name to load"
         }
     },
     "required": ["name"]
 }}
```

### 6.2 工具处理器注册

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
}
```

`load_skill` 的处理器直接调用 `SKILL_LOADER.get_content()`，将技能正文作为 `tool_result` 返回给 LLM。

---

## 7. 完整的工作流程

### 7.1 用户请求

```
用户：帮我审查这段代码
def calc(a,b,c,d):
    x=a+b
    y=c*d
    return x/y
```

### 7.2 Agent 处理流程

```
轮次 1：
  LLM 收到系统提示（包含技能目录 Layer 1）：
    Skills available:
      - code-review: Perform thorough code reviews...
      - pdf: Process PDF files...
      ...

  LLM：我需要使用 code-review 技能来审查代码
        tool_use: load_skill(name="code-review")

  工具结果（Layer 2 注入）：
    <skill name="code-review">
    # Code Review Skill

    ## Review Checklist
    ### 1. Security (Critical)
    - [ ] Injection vulnerabilities
    ### 2. Correctness
    - [ ] Logic errors
    ...（完整的审查清单和代码示例）
    </skill>

轮次 2：
  LLM：现在我有审查规范了，让我对照清单检查代码

  检查结果：
  1. [Correctness] 没有处理除零情况 → y 可能为 0，导致 ZeroDivisionError
  2. [Maintainability] 函数名 `calc` 不够具体 → 建议改为 `calculate_ratio`
  3. [Maintainability] 参数名需要类型注解
  4. [Security] 输入未验证 → 可能传入非数字类型

  返回详细审查意见
```

### 7.3 Token 消耗对比

```
场景：用户请求 code review，系统有 4 个技能

单层注入（全量加载）：
  系统提示 = ~200 (基础) + ~7200 (4 个技能完整内容) = ~7400 tokens
  每次调用固定成本：7400 tokens

两层注入（按需加载）：
  系统提示 = ~200 (基础) + ~80 (4 个技能的元数据) = ~280 tokens
  load_skill 返回 = ~2000 tokens (仅 code-review)
  总计 = ~2280 tokens

节省 = 7400 - 2280 = 5120 tokens (69.2%)
```

---

## 8. 高级特性

> **注意**：以下 8.1 节（技能依赖）和 8.2 节（技能参数）为扩展设计方向，当前源码（`agents/s05_skill_loading.py`）中未实现，仅作为设计参考。

### 8.1 技能依赖

> **注意**：以下特性为扩展设计方向，当前源码（`agents/s05_skill_loading.py`）中未实现。

某些技能可能依赖其他技能，可以在 frontmatter 中声明（需扩展 SkillLoader）：

```yaml
---
name: pull-request
description: 完整的 PR 工作流程
depends:
  - git
  - code-review
---
```

扩展 `get_content()` 以支持依赖加载：

```python
def get_content(self, name: str) -> str:
    skill = self.skills.get(name)
    content = skill['body']

    # 加载依赖
    for dep in skill["meta"].get("depends", []):
        dep_content = self.skills.get(dep, {}).get("body", "")
        content = f"<dependency name=\"{dep}\">{dep_content}</dependency>\n\n{content}"

    return f"<skill name=\"{name}\">\n{content}\n</skill>"
```

### 8.2 技能参数

> **注意**：以下特性为扩展设计方向，当前源码（`agents/s05_skill_loading.py`）中未实现。

某些技能可能需要参数化（需扩展工具定义和处理器）：

```yaml
---
name: test-framework
description: 测试框架助手
parameters:
  framework:
    type: string
    enum: [pytest, unittest, jest]
    default: pytest
---
```

### 8.3 辅助资源的使用

像 `agent-builder` 这样包含 `references/` 和 `scripts/` 的技能，可以在 SKILL.md 的正文中引用这些资源：

```markdown
## Resources

**Philosophy & Theory**:
- `references/agent-philosophy.md` - Deep dive into why agents work

**Implementation**:
- `references/minimal-agent.py` - Complete working agent (~80 lines)

**Scaffolding**:
- `scripts/init_agent.py` - Generate new agent projects
```

LLM 在需要时可以通过 `read_file` 工具进一步加载这些辅助资源，形成"三层加载"：元数据 -> 技能正文 -> 辅助资源。

---

## 9. 与 s04 的对比

| 方面 | s04 子 Agent | s05 技能系统 |
|------|-------------|--------------|
| 加载时机 | 调用时创建子对话 | 调用时加载静态知识 |
| 内容类型 | 动态推理结果 | 静态参考文档 |
| Token 成本 | 中等（子 Agent 的完整循环） | 低（一次性注入） |
| 适用场景 | 需要多步推理的子任务 | 需要规范/模板的参考知识 |
| 可复用性 | 每次重新推理 | 加载相同内容，可缓存 |
| 隔离性 | 完全隔离的上下文 | 注入到主上下文中 |
| 实现复杂度 | 需要独立的 Agent 循环 | 只需一个工具处理器 |

**关键区别**：子 Agent 擅长"做事情"，技能系统擅长"知道事情"。两者互补：先通过技能加载专业知识，再委托子 Agent 执行复杂任务。

---

## 10. 常见问题

### Q1：技能和提示工程有什么区别？

| 特性 | 提示工程 | 技能系统 |
|------|----------|----------|
| 存储 | 硬编码在代码中 | 独立的 Markdown 文件 |
| 更新 | 需要修改代码 | 编辑 SKILL.md |
| 分发 | 与代码绑定 | 独立分享、版本控制 |
| 加载方式 | 始终在系统提示中 | 按需加载到 tool_result |
| 可扩展性 | 修改代码 | 添加目录和文件 |

**答案**：技能系统是提示工程的"文件化"和"模块化"升级版。它将知识从代码中解耦，支持独立维护和按需加载。

### Q2：为什么不用数据库存储技能？

**答案**：
1. 文件系统更简单，不需要额外依赖
2. Markdown 便于编辑和版本控制（Git 友好）
3. 文件结构天然支持技能的组织（目录 = 技能，文件 = 内容）
4. 小规模场景下，数据库是过度设计
5. SKILL.md 的格式让非程序员也能编辑技能

### Q3：技能可以被 LLM 修改吗？

**答案**：技术上可以（Agent 有 write_file 工具），但不推荐。技能应该是"只读"的参考。如果需要动态内容，考虑：
1. 使用工具动态获取信息
2. 使用子 Agent 生成内容
3. 将技能作为模板，由子 Agent 填充

### Q4：frontmatter 解析器为什么不用 yaml 库？

**答案**：源码使用了简化的字符串分割解析，而非标准 yaml 库。原因可能是：
1. 减少外部依赖（无需 `pip install pyyaml`）
2. 对于简单的 key: value 格式，正则 + split 足够
3. 代价是不支持多行值（如 `description: |`）和复杂 YAML 结构

如果需要支持复杂 frontmatter，建议引入 `yaml.safe_load()` 替换当前实现。

---

## 11. 最佳实践

### 11.1 编写好的技能描述

```yaml
# 差：模糊的描述，LLM 无法判断何时使用
description: 这是一个关于代码的技能

# 好：明确的触发条件和用途
description: Perform thorough code reviews with security, performance, and maintainability analysis. Use when user asks to review code, check for bugs, or audit a codebase.
```

**原则**：描述中应包含触发条件（"Use when..."），让 LLM 能根据用户意图自行判断是否需要加载。

### 11.2 技能内容的组织

```markdown
---
name: my-skill
description: 简短描述（包含触发条件）
---

# 技能标题

## 快速开始
[3-5 句话说明如何使用]

## 核心概念
[关键概念解释]

## 步骤指南
[详细的操作步骤]

## 代码示例
[可直接使用的代码模板]

## 常见问题
[FAQ]
```

### 11.3 技能粒度

```
太粗（一个技能包含太多内容）：
- programming: 所有编程相关

太细（技能过于碎片化）：
- python-naming: Python 命名规范
- python-types: Python 类型注解
- python-docs: Python 文档字符串

合适（按功能域划分）：
- code-review: 代码审查规范（包含安全、性能、可维护性）
- pdf: PDF 处理（读取、创建、合并、拆分）
- agent-builder: Agent 构建方法论
```

### 11.4 避免多行 description

由于当前 frontmatter 解析器的限制，避免使用 YAML 管道语法：

```yaml
# 避免这样写（多行值不会被正确解析）：
description: |
  多行描述
  第二行

# 应该写成单行：
description: 简洁的单行描述，包含触发条件
```

---

## 12. 小结

### 12.1 核心要点

| 要点 | 说明 |
|------|------|
| **两层注入** | Layer 1 元数据常驻系统提示，Layer 2 完整内容按需加载 |
| **SKILL.md** | YAML frontmatter（name + description）+ Markdown 正文 |
| **SkillLoader** | 递归扫描、正则解析 frontmatter、字典存储 |
| **load_skill 工具** | 将技能正文以 `<skill>` 标签返回到 tool_result |
| **Token 效率** | 将成本从 O(N * T_body) 降至 O(N * T_meta + k * T_body) |

### 12.2 代码模板

```python
# 1. 初始化 SkillLoader（启动时扫描一次）
SKILL_LOADER = SkillLoader(SKILLS_DIR)

# 2. Layer 1：元数据嵌入系统提示
SYSTEM = f"""You are a coding agent.
Skills available:
{SKILL_LOADER.get_descriptions()}"""

# 3. Layer 2：通过工具按需加载
TOOLS.append({"name": "load_skill", ...})
TOOL_HANDLERS["load_skill"] = lambda **kw: SKILL_LOADER.get_content(kw["name"])

# 4. Agent 循环中无需特殊处理——load_skill 像其他工具一样工作
```

### 12.3 关键洞察

```
技能系统的核心思想：
1. 知识不应该常驻在上下文中——大部分知识大部分时间用不到
2. 系统提示只包含"目录"（Layer 1），不包含"内容"（Layer 2）
3. LLM 自主决定何时需要加载知识（通过调用 load_skill 工具）
4. 技能是可编辑、可版本控制的独立文件（skills/<name>/SKILL.md）

设计原则：
- 轻量元数据常驻（每技能 ~20 tokens）
- 重量内容按需加载（仅加载需要的技能）
- 技能独立于 Agent 代码（热更新无需重启）
- 文件系统即数据库（简单、可 Git 追踪）
```

---

## 13. 练习

### 练习 1：添加新技能

创建一个 `git-commit` 技能，定义好的 commit message 规范。要求：
1. 在 `skills/git-commit/` 下创建 SKILL.md
2. 包含 frontmatter（name + description）和正文
3. 运行程序验证技能被自动发现

### 练习 2：改进 Frontmatter 解析器

将 `_parse_frontmatter` 改为使用 `yaml.safe_load()`，使其正确支持多行 description。

### 练习 3：技能搜索

添加 `search_skills` 工具，允许 LLM 根据关键词在技能正文中搜索。

### 练习 4：技能验证

实现技能验证，检查 SKILL.md 的格式是否正确（必需字段是否存在、description 是否为空等）。

---

## 14. 下一步

现在 Agent 可以按需加载知识，但上下文仍会随对话增长。对于长对话，需要一种机制来清理旧内容。

在 [s06：上下文压缩](./s06-context-compact.md) 中，我们将学习：
- 三层压缩策略（微压缩、自动压缩、手动压缩）
- Token 估算与阈值触发机制
- 转录存档的 JSONL 格式与恢复机制

---

< 上一章：[s04-子 Agent](./s04-subagent.md) | [目录](./index.md) | 下一章：[s06-上下文压缩](./s06-context-compact.md) >
