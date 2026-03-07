# s05：技能系统 —— 按需加载知识

> **核心口号**：*"需要时加载知识，而非预先加载"*

> **学习目标**：理解知识注入的两种模式，掌握技能系统设计，学会高效的 Token 管理

---

## 学习目标

完成本章后，你将能够：

1. **理解两层注入模式** —— 元数据 vs 完整内容
2. **设计技能系统** —— SKILL.md 的结构规范
3. **实现按需加载** —— 只在需要时加载详细内容
4. **掌握技能注册** —— 自动发现和注册技能

---

## 0. 上手演练（建议先做）

先跑技能加载示例，再看两层注入实现：

```bash
uv run python agents/s05_skill_loading.py
```

建议观察：

1. 首轮系统提示是否只包含轻量技能目录。
2. 何时触发 `load_skill`，以及加载后上下文如何变化。
3. 只按需注入技能内容时，Token 消耗如何下降。

### 本章完成标准

- 能清楚区分 Layer 1 元数据与 Layer 2 完整内容的职责。
- 能独立新增一个 SKILL.md 并让系统自动发现。
- 能说明“按需加载”在上下文成本与响应质量上的收益。

---

## 1. 问题的本质

### 1.1 知识注入的困境

假设你希望 Agent 遵循多种专业规范：

```
需要的专业知识：
- Git 工作流程（2000 tokens）
- 代码审查规范（1500 tokens）
- 测试最佳实践（1800 tokens）
- 文档编写指南（1200 tokens）
- 安全编码标准（2500 tokens）
...
总计：9000+ tokens
```

**问题**：如果全部放入系统提示：

```python
SYSTEM = """你是一个编程助手。

Git 工作流程：
[2000 tokens 的详细说明]

代码审查规范：
[1500 tokens 的详细说明]

测试最佳实践：
[1800 tokens 的详细说明]

...
"""
```

**后果**：
1. 每次调用都发送 9000+ tokens
2. 大部分对话只用到一个技能（浪费）
3. 上下文窗口被静态内容占据

### 1.2 理想的方式

```
第一次调用（轻量）：
系统提示（100 tokens）：
  你是一个编程助手。
  可用技能：git, code-review, testing, docs, security

用户：帮我做一个 code review
  ↓
LLM 决定加载 code-review 技能

第二次调用（按需）：
系统提示（100 tokens）+ 工具结果（1500 tokens）：
  <skill name="code-review">
  [完整的代码审查规范]
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
│  技能 A：完整的详细内容（2000t）    │ ← 大部分时间不需要
│  技能 B：完整的详细内容（1500t）    │
│  技能 C：完整的详细内容（1800t）    │
│  技能 D：完整的详细内容（1200t）    │
│  ...                                │
└─────────────────────────────────────┘
每次调用：6500+ tokens 的固定成本

✅ 两层注入（高效）：
┌─────────────────────────────────────┐
│    系统提示（每次调用，轻量）        │
│                                     │
│  可用技能：                          │
│    - git: Git 工作流程助手          │ ← 只占 ~100 tokens
│    - code-review: 代码审查规范       │
│    - testing: 测试最佳实践          │
│    - docs: 文档编写指南             │
│                                     │
│  用户需要时，通过 load_skill 工具   │
│  加载完整内容                       │
└─────────────────────────────────────┘
第一次调用：~100 tokens
需要时加载：+1500-2000 tokens（仅一次）
```

### 2.2 两层的内容设计

| 层级 | 内容 | Token 估算 | 何时加载 |
|------|------|------------|----------|
| **Layer 1（元数据）** | 技能名称 + 简短描述 | ~100/skill | 每次 LLM 调用 |
| **Layer 2（完整内容）** | 详细的规范、步骤、示例 | ~1500-2000/skill | LLM 主动请求时 |

**Layer 1 示例**（系统提示）：
```
可用技能：
  - git: Git 工作流程和版本控制助手
  - code-review: 代码审查规范和检查清单
  - testing: 单元测试和集成测试最佳实践
```

**Layer 2 示例**（按需加载）：
```xml
<skill name="code-review">
# 代码审查规范

## 1. 功能性
- 代码是否实现了需求？
- 边界条件是否处理？
- 错误处理是否完善？

## 2. 可读性
- 变量命名是否清晰？
- 函数是否过于复杂？
- 是否有必要的注释？

## 3. 性能
- 是否有明显性能问题？
- 资源使用是否合理？

...（1500+ tokens 的详细内容）
</skill>
```

---

## 3. 技能文件结构

### 3.1 SKILL.md 格式

每个技能是一个目录，包含 `SKILL.md` 文件：

```
skills/
├── code-review/
│   └── SKILL.md          # 技能定义
├── git/
│   └── SKILL.md
├── testing/
│   └── SKILL.md
└── pdf/
    ├── SKILL.md
    ├── scripts/          # 可选：辅助脚本
    │   └── rotate.py
    └── references/       # 可选：参考资料
        └── examples.md
```

### 3.2 SKILL.md 的结构

```markdown
---
name: code-review
description: 代码审查助手，提供审查规范和检查清单
---

# 代码审查技能

## 使用场景

当代码需要审查时，使用本技能：
- Pull Request 审查
- 代码质量检查
- 安全性评估

## 审查规范

### 1. 功能性
- [ ] 代码实现了需求
- [ ] 边界条件已处理
...

### 2. 可读性
- [ ] 命名清晰
...

## 检查清单

- 使用有意义的变量名
- 函数长度 < 50 行
- 圈复杂度 < 10
...
```

**关键点**：
1. **YAML 前置元数据**（`---` 之间）：name 和 description
2. **Markdown 正文**：详细的技能内容
3. **可选资源**：scripts、references 等

---

## 4. SkillLoader 实现

### 4.1 核心类

```python
import re
from pathlib import Path
from typing import Dict, List

class SkillLoader:
    def __init__(self, skills_dir: Path):
        self.skills_dir = skills_dir
        self.skills: Dict[str, Dict] = {}

        # 自动发现所有 SKILL.md
        self._discover_skills()

    def _discover_skills(self):
        """扫描目录，加载所有技能"""
        for skill_file in sorted(self.skills_dir.rglob("SKILL.md")):
            text = skill_file.read_text(encoding="utf-8")
            meta, body = self._parse_frontmatter(text)

            # 使用目录名作为技能 ID
            skill_name = meta.get("name", skill_file.parent.name)
            self.skills[skill_name] = {
                "meta": meta,
                "body": body,
                "path": skill_file
            }

    def _parse_frontmatter(self, text: str) -> tuple:
        """解析 YAML 前置元数据"""
        match = re.match(r'^---\n(.*?)\n---\n(.*)$', text, re.DOTALL)
        if match:
            import yaml
            meta = yaml.safe_load(match.group(1)) or {}
            body = match.group(2)
        else:
            meta = {}
            body = text
        return meta, body

    def get_descriptions(self) -> str:
        """获取 Layer 1：技能名称和描述"""
        lines = ["可用技能："]
        for name, skill in sorted(self.skills.items()):
            desc = skill["meta"].get("description", "")
            lines.append(f"  - {name}: {desc}")
        return "\n".join(lines)

    def get_content(self, name: str) -> str:
        """获取 Layer 2：完整技能内容"""
        skill = self.skills.get(name)
        if not skill:
            return f"错误：未知技能 '{name}'"

        return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

### 4.2 使用示例

```python
# 初始化
SKILL_LOADER = SkillLoader(Path("skills"))

# Layer 1：系统提示
SYSTEM = f"""你是一个编程助手。

{SKILL_LOADER.get_descriptions()}

当需要使用某个技能时，调用 load_skill 工具。"""

# Layer 2：按需加载
TOOLS.append({
    "name": "load_skill",
    "description": "加载指定技能的详细内容",
    "input_schema": {
        "type": "object",
        "properties": {
            "name": {
                "type": "string",
                "description": "技能名称"
            }
        },
        "required": ["name"]
    }
})

TOOL_HANDLERS["load_skill"] = lambda **kw: SKILL_LOADER.get_content(kw["name"])
```

---

## 5. 完整的工作流程

### 5.1 用户请求

```
用户：帮我审查这段代码
def calc(a,b,c,d):
    x=a+b
    y=c*d
    return x/y
```

### 5.2 Agent 处理流程

```
轮次 1：
  LLM 收到系统提示（包含技能列表）：
    可用技能：
      - code-review: 代码审查助手
      - git: Git 工作流程
      ...

  LLM：我需要使用 code-review 技能
        tool_use: load_skill(name="code-review")

  工具结果：
    <skill name="code-review">
    # 代码审查规范

    ## 1. 功能性
    - 边界条件处理
    - 错误处理

    ## 2. 可读性
    - 命名规范
    - 函数复杂度
    ...
    </skill>

轮次 2：
  LLM：现在我有审查规范了，让我检查代码

  检查结果：
  1. 函数名 `calc` 不够具体 → 建议改为 `calculate_ratio`
  2. 参数名需要类型注解
  3. 没有处理除零情况 → 需要添加检查
  4. 变量命名太简短

  返回详细审查意见
```

---

## 6. 高级特性

### 6.1 技能依赖

某些技能可能依赖其他技能：

```yaml
---
name: pull-request
description: 完整的 PR 工作流程
depends:
  - git
  - code-review
---
```

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

### 6.2 技能参数

某些技能可能需要参数化：

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

使用时：
```
tool_use: load_skill(name="test-framework", framework="pytest")
```

### 6.3 动态技能内容

技能可以包含 Python 代码块，在加载时执行：

```markdown
---
name: project-stats
description: 项目统计
---
import os
from pathlib import Path

def get_file_count():
    return len(list(Path(".").rglob("*.py")))

def get_line_count():
    total = 0
    for f in Path(".").rglob("*.py"):
        total += len(f.read_text().splitlines())
    return total

# 项目统计
- Python 文件数：{{get_file_count()}}
- 总行数：{{get_line_count()}}
```

---

## 7. 与 s04 的对比

| 方面 | s04 子 Agent | s05 技能系统 |
|------|-------------|--------------|
| 加载时机 | 调用时创建 | 调用时加载 |
| 内容类型 | 动态推理 | 静态知识 |
| Token 成本 | 中等（子 Agent 循环） | 低（一次性加载） |
| 适用场景 | 需要多步推理 | 需要规范/模板 |
| 可复用性 | 每次重新生成 | 加载相同内容 |

---

## 8. 常见问题

### Q1：技能和提示工程有什么区别？

| 特性 | 提示工程 | 技能系统 |
|------|----------|----------|
| 存储 | 硬编码在代码中 | 独立的 Markdown 文件 |
| 更新 | 需要修改代码 | 编辑 SKILL.md |
| 分发 | 与代码绑定 | 独立分享 |
| 版本控制 | 与代码一起 | 独立版本 |

**答案**：技能系统是提示工程的"文件化"和"模块化"。

### Q2：为什么不用数据库存储技能？

**答案**：
1. 文件系统更简单，不需要额外依赖
2. Markdown 便于编辑和版本控制
3. 文件结构天然支持技能的组织
4. 小规模场景下，数据库是过度设计

### Q3：技能可以被 LLM 修改吗？

**答案**：技术上可以，但不推荐。技能应该是"只读"的参考。如果需要动态内容，考虑：
1. 使用工具动态获取信息
2. 使用子 Agent 生成内容
3. 将技能作为模板，由子 Agent 填充

---

## 9. 最佳实践

### 9.1 编写好的技能描述

```yaml
# ❌ 模糊的描述
description: 这是一个关于代码的技能

# ✅ 明确的描述
description: 代码审查助手，提供功能性、可读性、性能三个维度的审查清单。适用于 PR 审查、代码质量检查、安全性评估等场景。
```

### 9.2 技能内容的组织

```markdown
---
name: my-skill
description: 简短描述
---

## 快速开始
[3-5 句话说明如何使用]

## 核心概念
[关键概念解释]

## 步骤指南
[详细的操作步骤]

## 常见问题
[FAQ]

## 参考资源
[链接到更多信息]
```

### 9.3 技能粒度

```
太粗：
- programming: 所有编程相关

太细：
- python-naming: Python 命名规范
- python-types: Python 类型注解
- python-docs: Python 文档字符串

合适：
- python-style: Python 代码风格（包含命名、类型、文档）
- code-review: 代码审查规范
- testing: 测试最佳实践
```

---

## 10. 小结

### 10.1 核心要点

| 要点 | 说明 |
|------|------|
| **两层注入** | 元数据（系统提示）+ 完整内容（按需） |
| **SKILL.md** | YAML 前置元数据 + Markdown 正文 |
| **按需加载** | 只在 LLM 请求时加载详细内容 |
| **Token 效率** | 大幅减少不必要的知识传输 |

### 10.2 代码模板

```python
# 初始化
SKILL_LOADER = SkillLoader(Path("skills"))

# 系统提示（Layer 1）
SYSTEM = f"""{SKILL_LOADER.get_descriptions()}"""

# 工具定义
TOOLS.append({"name": "load_skill", ...})

# 处理器
TOOL_HANDLERS["load_skill"] = lambda **kw: SKILL_LOADER.get_content(kw["name"])
```

### 10.3 关键洞察

```
技能系统的核心思想：
1. 知识不应该常驻在上下文中
2. 只在需要时加载详细的参考信息
3. 系统提示只包含"目录"，不包含"内容"
4. 技能是可编辑、可版本控制的独立文件

设计原则：
- 轻量元数据常驻
- 重量内容按需加载
- 技能独立于 Agent 代码
- 文件系统即数据库
```

---

## 11. 练习

### 练习 1：添加新技能

创建一个 `git-commit` 技能，定义好的 commit message 规范。

### 练习 2：技能继承

实现技能继承机制，子技能可以扩展父技能的内容。

### 练习 3：技能搜索

添加 `search_skills` 工具，允许 LLM 根据关键词搜索相关技能。

### 练习 4：技能验证

实现技能验证，检查 SKILL.md 的格式是否正确。

---

## 12. 下一步

现在 Agent 可以按需加载知识，但上下文仍会随对话增长。对于长对话，需要一种机制来清理旧内容。

在 [s06：上下文压缩](./s06-context-compact.md) 中，我们将学习：
- 三层压缩策略
- 微压缩、自动压缩、手动压缩
- 转录存档机制

---

< 上一章：[s04-子 Agent](./s04-subagent.md) | [目录](./index.md) | 下一章：[s06-上下文压缩](./s06-context-compact.md) >
