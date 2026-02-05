# Context.py 详细解析

## 📋 目录
1. [概述](#概述)
2. [架构流程图](#架构流程图)
3. [类结构详解](#类结构详解)
4. [Prompt 构建详解](#prompt-构建详解)
5. [设计亮点](#设计亮点)

---

## 概述

`context.py` 是 nanobot 的 **Prompt 构建器**，负责将各种信息源（身份、配置、记忆、技能、历史记录）组装成完整的 LLM 输入提示。

**核心职责：**
- 📝 构建系统提示词（System Prompt）
- 💬 构建完整消息列表（Messages）
- 🎨 处理多媒体内容（图片 Base64 编码）
- 🔧 支持工具调用和对话流

---

## 架构流程图

### 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ContextBuilder                              │
│                                                                      │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐            │
│  │ MemoryStore  │   │ SkillsLoader │   │  Bootstrap   │            │
│  │  (记忆系统)   │   │  (技能加载器) │   │   Files      │            │
│  └──────────────┘   └──────────────┘   └──────────────┘            │
│         │                   │                   │                    │
│         └───────────────────┼───────────────────┘                    │
│                             ▼                                        │
│                    ┌─────────────────┐                               │
│                    │ build_messages()│                               │
│                    └─────────────────┘                               │
│                             │                                        │
│                             ▼                                        │
│                    ┌─────────────────┐                               │
│                    │ build_system_   │                               │
│                    │   prompt()      │                               │
│                    └─────────────────┘                               │
└─────────────────────────────────────────────────────────────────────┘
```

### Prompt 构建完整流程

```
用户请求
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│ build_messages(                                               │
│   history, current_message, skill_names,                     │
│   media, channel, chat_id                                     │
│ )                                                            │
└──────────────────────────────────────────────────────────────┘
    │
    ├──────────────────────────────────────────────────────────┤
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│ build_system_prompt(skill_names)                             │
│   构建 System Prompt                                          │
└──────────────────────────────────────────────────────────────┘
    │
    ├──────────────────────────────────────────────────────────┤
    │                                                          │
    ▼                                                          ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────┐
│   Identity   │  │  Bootstrap   │  │    Memory    │  │Skills│
│  (身份信息)   │  │  Files       │  │  (记忆系统)   │  │     │
│  [1]         │  │  [2]         │  │  [3]         │  │[4]  │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──┬───┘
       │                 │                 │             │
       └─────────────────┼─────────────────┴─────────────┘
                         │
                         ▼
              ┌────────────────────┐
              │  拼接所有部分      │
              │  (--- 分隔)        │
              └────────────────────┘
                         │
                         ▼
              ┌────────────────────┐
              │ System Prompt ✅   │
              └────────────────────┘
    │
    ├──────────────────────────────────────────────────────────┤
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│ 构建消息列表:                                                  │
│                                                              │
│  messages = [                                                 │
│    {role: "system", content: system_prompt + session_info},  │
│    ...history...,                                            │
│    {role: "user", content: current_message (+ images)}        │
│  ]                                                           │
└──────────────────────────────────────────────────────────────┘
    │
    ▼
  完整 Prompt 📤
```

### System Prompt 构建细节流程

```
build_system_prompt(skill_names=None)
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│ parts = []                                                │
└──────────────────────────────────────────────────────────┘
    │
    ├─────────────────┬─────────────────┬──────────────────┤
    │                 │                 │                  │
    ▼                 ▼                 ▼                  ▼
┌─────────┐   ┌───────────┐   ┌─────────────┐   ┌─────────────┐
│Identity │   │Bootstrap  │   │  Memory     │   │   Skills    │
│         │   │Files      │   │             │   │             │
│         │   │           │   │ - Long-term │   │ - Always    │
│- nanobot│   │ - AGENTS.md│   │ - Today     │   │   Skills    │
│  介绍   │   │ - SOUL.md │   │             │   │             │
│- 时间   │   │ - USER.md │   └─────────────┘   │ - Summary   │
│- 工作区 │   │ - TOOLS.md│                      │   (可用)    │
│- 行为   │   │ - IDENTITY│                      │             │
│  准则   │   │   .md     │                      │             │
└────┬────┘   └─────┬─────┘                      └──────┬──────┘
     │              │                                   │
     └──────────────┼───────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │ 用 "\n\n---\n\n" 拼接│
         │ 所有非空部分         │
         └──────────────────────┘
                    │
                    ▼
          完整 System Prompt 🎯
```

### Skills 处理流程

```
SkillsLoader
    │
    ├────────────────────────────────────────────────────┐
    │                                                    │
    ▼                                                    ▼
┌─────────────────────┐                   ┌──────────────────────┐
│ list_skills()       │                   │ get_always_skills()  │
│                     │                   │                      │
│ - workspace/skills/ │                   │ - 筛选 always=true   │
│   (用户自定义)       │                   │ - 检查依赖           │
│ - nanobot/skills/   │                   │ - 返回完整内容       │
│   (内置技能)         │                   │                      │
└──────────┬──────────┘                   └──────────┬───────────┘
           │                                          │
           └─────────────────┬────────────────────────┘
                             │
                             ▼
              ┌─────────────────────────────┐
              │ 1. Always Skills:          │
              │    - 直接加载完整内容        │
              │    - 放入 System Prompt     │
              │                            │
              │ 2. Other Skills:           │
              │    - 只显示摘要 (XML)       │
              │    - 需要时用 read_file    │
              └─────────────────────────────┘
```

### 消息构建流程

```
build_messages(history, current_message, media)
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ messages = []                                       │
└─────────────────────────────────────────────────────┘
    │
    ├─────────────────────────────────────────────────┤
    │                                                  │
    ▼                                                  ▼
┌──────────────────────┐               ┌──────────────────┐
│ System Message       │               │ User Message     │
│ {                   │               │ {                │
│   role: "system",   │               │   role: "user",  │
│   content:          │               │   content:       │
│     [1] Identity     │               │     [text]       │
│     [2] Bootstrap    │               │     [+images]    │
│     [3] Memory       │               │ }                │
│     [4] Skills       │               └──────────────────┘
│     [+session]       │                       │
│ }                   │                       ▼
└──────────┬───────────┘             _build_user_content()
           │                           │
           │                    ┌──────┴──────┐
           │                    ▼             ▼
           │              有图片?          无图片
           │                │                 │
           │            是│                  │
           │              ▼                  │
           │     ┌────────────────┐          │
           │     │ Base64 编码     │          │
           │     │ 图片文件        │          │
           │     └────────┬───────┘          │
           │              │                  │
           │              └──────┬───────────┘
           │                     ▼
           │              返回内容数组
           │
           ▼
     历史消息直接追加
     ...history...
           │
           ▼
    完整消息列表 📤
```

---

## 类结构详解

### 1. ContextBuilder 类

```python
class ContextBuilder:
    """
    构建 Agent 的上下文（System Prompt + Messages）
    
    组装引导文件、记忆、技能和对话历史，
    为 LLM 提供连贯的提示。
    """
```

**核心属性：**

```python
BOOTSTRAP_FILES = [
    "AGENTS.md",  # Agent 配置
    "SOUL.md",    # Agent 灵魂/个性
    "USER.md",    # 用户偏好
    "TOOLS.md",   # 工具说明
    "IDENTITY.md" # 身份定义
]

def __init__(self, workspace: Path):
    self.workspace = workspace        # 工作区路径
    self.memory = MemoryStore(workspace)  # 记忆系统
    self.skills = SkillsLoader(workspace)  # 技能加载器
```

**关键方法：**

| 方法 | 功能 | 输入 | 输出 |
|------|------|------|------|
| `build_system_prompt()` | 构建系统提示词 | skill_names | 完整 System Prompt |
| `build_messages()` | 构建完整消息列表 | history, message, media, etc. | LLM 调用格式 |
| `add_tool_result()` | 添加工具执行结果 | messages, tool_call_id, result | 更新的消息列表 |
| `add_assistant_message()` | 添加助手回复 | messages, content, tool_calls | 更新的消息列表 |

---

## Prompt 构建详解

### 🎯 核心思想

Prompt 构建采用**分层组装**策略：

1. **Identity Layer**: 基础身份信息（始终存在）
2. **Bootstrap Layer**: 配置文件（按需加载）
3. **Memory Layer**: 记忆系统（长期 + 今天）
4. **Skills Layer**: 技能（Always 完整 + 其他摘要）
5. **Session Layer**: 当前会话信息（channel, chat_id）

### 📝 System Prompt 组装顺序

```python
def build_system_prompt(self, skill_names: list[str] | None = None) -> str:
    parts = []

    # [1] Core Identity - 必须有
    parts.append(self._get_identity())

    # [2] Bootstrap Files - 如果存在
    bootstrap = self._load_bootstrap_files()
    if bootstrap:
        parts.append(bootstrap)

    # [3] Memory Context - 如果有记忆
    memory = self.memory.get_memory_context()
    if memory:
        parts.append(f"# Memory\n\n{memory}")

    # [4] Skills - Progressive Loading
    # 4a. Always Skills: 完整内容
    always_skills = self.skills.get_always_skills()
    if always_skills:
        always_content = self.skills.load_skills_for_context(always_skills)
        if always_content:
            parts.append(f"# Active Skills\n\n{always_content}")

    # 4b. Available Skills: XML 摘要（按需加载）
    skills_summary = self.skills.build_skills_summary()
    if skills_summary:
        parts.append(f"""# Skills

The following skills extend your capabilities. To use a skill, read its SKILL.md file using the read_file tool.
Skills with available="false" need dependencies installed first - you can try installing them with apt/brew.

{skills_summary}""")

    return "\n\n---\n\n".join(parts)
```

### 🔍 各部分详细说明

#### [1] Identity - 核心身份

```python
def _get_identity(self) -> str:
    from datetime import datetime
    now = datetime.now().strftime("%Y-%m-%d %H:%M (%A)")
    workspace_path = str(self.workspace.expanduser().resolve())

    return f"""# nanobot 🐈

You are nanobot, a helpful AI assistant. You have access to tools that allow you to:
- Read, write, and edit files
- Execute shell commands
- Search the web and fetch web pages
- Send messages to users on chat channels
- Spawn subagents for complex background tasks

## Current Time
{now}

## Workspace
Your workspace is at: {workspace_path}
- Memory files: {workspace_path}/memory/MEMORY.md
- Daily notes: {workspace_path}/memory/YYYY-MM-DD.md
- Custom skills: {workspace_path}/skills/{{skill-name}}/SKILL.md

IMPORTANT: When responding to direct questions or conversations, reply directly with your text response.
Only use the 'message' tool when you need to send a message to a specific chat channel (like WhatsApp).
For normal conversation, just respond with text - do not call the message tool.

Always be helpful, accurate, and concise. When using tools, explain what you're doing.
When remembering something, write to {workspace_path}/memory/MEMORY.md"""
```

**设计要点：**
- ⏰ **动态时间**: 每次重新构建都会更新，避免时间幻觉
- 📁 **明确路径**: 清晰告诉 Agent 工作区位置
- 🎯 **行为规范**: 明确说明何时使用 message 工具
- 💾 **记忆指导**: 指导 Agent 如何持久化信息

#### [2] Bootstrap Files - 配置引导

```python
def _load_bootstrap_files(self) -> str:
    parts = []

    for filename in self.BOOTSTRAP_FILES:
        file_path = self.workspace / filename
        if file_path.exists():
            content = file_path.read_text(encoding="utf-8")
            parts.append(f"## {filename}\n\n{content}")

    return "\n\n".join(parts) if parts else ""
```

**文件说明：**

| 文件 | 用途 | 示例内容 |
|------|------|---------|
| `AGENTS.md` | Agent 配置规则 | 默认模型、最大 Token、工具配置 |
| `SOUL.md` | Agent 个性/风格 | 说话风格、偏好、禁忌 |
| `USER.md` | 用户偏好 | 工作习惯、通信方式、常做任务 |
| `TOOLS.md` | 工具使用说明 | 特定工具的使用场景 |
| `IDENTITY.md` | 额外身份定义 | 角色、职责、约束 |

**设计亮点：**
- 📂 **按需加载**: 只加载存在的文件
- 🏷️ **清晰标题**: 每个文件有 `## filename` 标题
- 🔄 **灵活配置**: 用户可以通过修改这些文件调整 Agent 行为

#### [3] Memory - 记忆系统

```python
# MemoryStore.get_memory_context()
def get_memory_context(self) -> str:
    parts = []

    # Long-term memory (MEMORY.md)
    long_term = self.read_long_term()
    if long_term:
        parts.append("## Long-term Memory\n" + long_term)

    # Today's notes (YYYY-MM-DD.md)
    today = self.read_today()
    if today:
        parts.append("## Today's Notes\n" + today)

    return "\n\n".join(parts) if parts else ""
```

**记忆层次：**

```
Memory/
├── MEMORY.md              # 长期记忆（永久）
│   └── 用户偏好、重要事实
└── 2025-02-06.md         # 今日记忆（日期文件）
    └── 今天的对话、任务
```

**设计亮点：**
- 🎯 **时间分离**: 长期记忆 vs 临时笔记
- 📅 **自动日期**: 每日笔记自动按日期归档
- 💡 **上下文聚焦**: 只提供长期记忆 + 今天，避免信息过载

#### [4] Skills - 技能系统（重点！）

**Progressive Loading 策略：**

```python
# 策略 1: Always Skills - 完整加载
always_skills = self.skills.get_always_skills()
# 例如: ["github", "weather"]
# 这些技能的完整内容会直接放入 System Prompt

# 策略 2: Available Skills - 摘要加载
skills_summary = self.skills.build_skills_summary()
# XML 格式摘要，只包含名称、描述、路径、可用性
# Agent 可以用 read_file 按需加载
```

**XML 摘要格式：**

```xml
<skills>
  <skill available="true">
    <name>github</name>
    <description>GitHub operations - clone, push, PR management</description>
    <location>/path/to/skills/github/SKILL.md</location>
  </skill>
  <skill available="false">
    <name>tmux</name>
    <description>Terminal multiplexer automation</description>
    <location>/path/to/skills/tmux/SKILL.md</location>
    <requires>CLI: tmux</requires>
  </skill>
</skills>
```

**技能加载优先级：**

```
1. workspace/skills/{name}/SKILL.md  (最高优先级 - 用户自定义)
2. nanobot/skills/{name}/SKILL.md     (备用 - 内置技能)
```

**Skill Metadata 示例：**

```markdown
---
name: github
description: GitHub operations - clone, push, PR management
always: true  # 标记为 always，会自动加载
requires:
  bins: ["git"]
  env: ["GITHUB_TOKEN"]
metadata: '{"nanobot": {"always": true, "requires": {"bins": ["git"]}}}'
---

# GitHub Skill

这里写技能的详细说明...
```

**设计亮点：**

1. **Token 优化** ✨
   - 常用技能（`always=true`）完整加载
   - 其他技能只加载摘要，按需读取
   - 避免不必要技能占用上下文

2. **依赖检查** 🔧
   - 自动检查命令行工具（`bins`）
   - 检查环境变量（`env`）
   - 不可用技能标记 `available="false"`

3. **渐进式加载** 📊
   ```
   用户请求: "帮我创建一个 GitHub PR"
                    ↓
   Agent 看到摘要中 github 可用
                    ↓
   调用 read_file("skills/github/SKILL.md")
                    ↓
   加载完整技能内容
   ```

4. **用户扩展性** 🚀
   - 用户可在 `workspace/skills/` 自定义技能
   - 会覆盖同名内置技能
   - 无需修改源码

### 🎨 User Message 构建

```python
def _build_user_content(self, text: str, media: list[str] | None) -> str | list[dict[str, Any]]:
    """构建带图片的用户消息"""
    if not media:
        return text

    images = []
    for path in media:
        p = Path(path)
        mime, _ = mimetypes.guess_type(path)
        if not p.is_file() or not mime or not mime.startswith("image/"):
            continue
        # Base64 编码
        b64 = base64.b64encode(p.read_bytes()).decode()
        images.append({
            "type": "image_url",
            "image_url": {"url": f"data:{mime};base64,{b64}"}
        })

    if not images:
        return text
    return images + [{"type": "text", "text": text}]
```

**多模态输出：**

```python
# 纯文本
"帮我分析这张图"

# 文本 + 图片
[
  {
    "type": "image_url",
    "image_url": {
      "url": "data:image/png;base64,iVBORw0KGgoAAAANS..."
    }
  },
  {
    "type": "text",
    "text": "帮我分析这张图"
  }
]
```

### 🔄 工具调用消息流

```python
# 1. Assistant 调用工具
assistant_msg = {
  "role": "assistant",
  "content": "让我帮你读取文件...",
  "tool_calls": [
    {
      "id": "call_abc123",
      "type": "function",
      "function": {
        "name": "read_file",
        "arguments": '{"path": "README.md"}'
      }
    }
  ]
}

# 2. 返回工具结果
tool_result_msg = {
  "role": "tool",
  "tool_call_id": "call_abc123",
  "name": "read_file",
  "content": "# README\n\n这是文件内容..."
}

# 3. Assistant 继续回复
final_msg = {
  "role": "assistant",
  "content": "文件内容如下：..."
}
```

---

## 设计亮点

### 1. 🎯 分层组装

```
System Prompt
├── Identity (基础层 - 必需)
├── Bootstrap (配置层 - 按需)
├── Memory (记忆层 - 按需)
└── Skills (能力层 - 渐进式)
```

### 2. 📦 Progressive Loading

- **Always Skills**: 自动加载完整内容
- **Available Skills**: 只加载 XML 摘要
- **On-Demand**: Agent 用 `read_file` 按需加载

**Token 优化效果：**
```
传统方式: 加载所有技能 → 10,000+ tokens (浪费)
Progressive: Always + 摘要 → 1,500 tokens (节省 85%)
```

### 3. 🔧 灵活配置

**用户只需在 workspace 创建文件：**

```
workspace/
├── AGENTS.md      # 自定义 Agent 配置
├── SOUL.md        # 定义 AI 个性
├── USER.md        # 记录偏好
├── skills/        # 自定义技能
│   └── myskill/
│       └── SKILL.md
└── memory/
    ├── MEMORY.md  # 长期记忆
    └── 2025-02-06.md
```

### 4. 🌐 多模态支持

- 自动识别图片类型（MIME）
- Base64 编码内联
- 支持文本 + 图片混合

### 5. 🧠 记忆分层

```
Long-term (MEMORY.md)    ← 永久重要信息
Today (YYYY-MM-DD.md)    ← 临时任务记录
Recent (最近 N 天)        ← 可选扩展
```

### 6. 🚀 可扩展性

**添加新技能：**

```bash
# 1. 创建技能目录
mkdir workspace/skills/mytool

# 2. 编写技能定义
cat > workspace/skills/mytool/SKILL.md <<EOF
---
name: mytool
description: My custom tool description
always: false
---

# My Tool Usage

这里写详细的使用说明...
EOF

# 3. 下次对话自动生效 ✨
```

---

## 实际例子

### 示例：完整 System Prompt

```markdown
# nanobot 🐈

You are nanobot, a helpful AI assistant. You have access to tools that allow you to:
- Read, write, and edit files
- Execute shell commands
- Search the web and fetch web pages
- Send messages to users on chat channels
- Spawn subagents for complex background tasks

## Current Time
2025-02-06 01:15 (Friday)

## Workspace
Your workspace is at: /home/user/workspace
- Memory files: /home/user/workspace/memory/MEMORY.md
- Daily notes: /home/user/workspace/memory/YYYY-MM-DD.md
- Custom skills: /home/user/workspace/skills/{skill-name}/SKILL.md

IMPORTANT: When responding to direct questions or conversations, reply directly with your text response.
Only use the 'message' tool when you need to send a message to a specific chat channel (like WhatsApp).
For normal conversation, just respond with text - do not call the message tool.

Always be helpful, accurate, and concise. When using tools, explain what you're doing.
When remembering something, write to /home/user/workspace/memory/MEMORY.md

---

## AGENTS.md

```markdown
Default model: anthropic/claude-3-5-sonnet
Max tokens: 4096
Temperature: 0.7
```

---

## SOUL.md

```markdown
你是 nanobot，一个友好的 AI 助手。
- 说话简洁明了
- 善欢用 emoji 表达情感
- 乐于助人但不过度热情
```

---

# Memory

## Long-term Memory
- 用户是开发者，主要使用 Python
- 工作在 UTC+8 时区
- 喜欢用 Vim 编辑器

## Today's Notes
- 10:00: 查看项目结构
- 12:30: 午餐休息
- 14:00: 继续开发

---

# Active Skills

### Skill: github

GitHub 操作技能，包括：
- 克隆仓库
- 创建 Pull Request
- 管理 Issues
- 查看代码

---

# Skills

The following skills extend your capabilities. To use a skill, read its SKILL.md file using the read_file tool.
Skills with available="false" need dependencies installed first - you can try installing them with apt/brew.

<skills>
  <skill available="true">
    <name>github</name>
    <description>GitHub operations - clone, push, PR management</description>
    <location>/home/user/skills/github/SKILL.md</location>
  </skill>
  <skill available="true">
    <name>weather</name>
    <description>Get weather information for any location</description>
    <location>/home/user/skills/weather/SKILL.md</location>
  </skill>
  <skill available="false">
    <name>tmux</name>
    <description>Terminal multiplexer automation</description>
    <location>/home/user/skills/tmux/SKILL.md</location>
    <requires>CLI: tmux</requires>
  </skill>
</skills>
```

---

## 总结

**ContextBuilder 的核心价值：**

1. ✅ **完整上下文**: 聚合所有必要信息
2. ✅ **Token 高效**: Progressive Loading 策略
3. ✅ **灵活配置**: 用户可自定义所有行为
4. ✅ **易于扩展**: 添加技能、记忆、配置都很容易
5. ✅ **多模态**: 支持文本 + 图片

**Prompt 构建关键点：**

```
分层组装 + 渐进加载 + 按需扩展
```

这个设计让 nanobot 能够在**保持轻量**的同时，提供**强大的上下文管理能力**。
