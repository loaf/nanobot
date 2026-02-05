# Nanobot 项目全面分析报告

**生成时间**: 2026-02-05
**分析目的**: 系统性学习 AI 架构及编程知识
**项目版本**: nanobot v0.1.3.post4
**代码规模**: ~5,302 行 Python 代码 + ~44 个 Python 文件

---

## 📋 目录

1. [项目概览](#项目概览)
2. [核心架构模式](#核心架构模式)
3. [模块深度分析](#模块深度分析)
   - [Agent 循环系统](#agent-循环系统)
   - [Provider 抽象层](#provider-抽象层)
   - [Tool 系统](#tool-系统)
   - [Session 管理](#session-管理)
   - [Memory 系统](#memory-系统)
   - [Skills 系统](#skills-系统)
   - [Subagent 管理](#subagent-管理)
   - [Communication Channels](#communication-channels)
   - [Message Bus](#message-bus)
   - [Cron 调度系统](#cron-调度系统)
   - [Configuration 系统](#configuration-系统)
   - [CLI 架构](#cli-架构)
4. [设计决策与最佳实践](#设计决策与最佳实践)
5. [学习路径](#学习路径)

---

## 项目概览

### 🎯 项目定位
Nanobot 是一个超轻量级个人 AI 助手框架，灵感来自 [Clawdbot](https://github.com/openclaw/openclaw)，但代码量减少了 99%（从 430k+ 行降至 ~4k 行）。

### 📦 技术栈
- **核心语言**: Python 3.11+
- **异步框架**: asyncio
- **配置管理**: Pydantic
- **CLI 框架**: Typer
- **日志**: Loguru
- **消息格式**: OpenAI Function Calling 兼容
- **Bridge**: Node.js (用于 WhatsApp 集成)

### 🏗️ 项目结构

```
nanobot/
├── agent/              # 核心 Agent 系统
│   ├── loop.py          # Agent 循环引擎
│   ├── context.py       # Prompt 构建器
│   ├── memory.py        # 持久化记忆
│   ├── skills.py        # 技能加载器
│   ├── subagent.py     # 子 Agent 管理器
│   └── tools/           # 工具系统
│       ├── base.py       # 工具基类
│       ├── registry.py    # 工具注册表
│       ├── filesystem.py  # 文件系统工具
│       ├── shell.py       # Shell 执行工具
│       ├── web.py        # Web 搜索/抓取工具
│       ├── message.py    # 消息发送工具
│       ├── spawn.py      # 子 Agent 生成工具
│       └── cron.py       # 调度任务工具
├── providers/          # LLM Provider 抽象层
│   ├── base.py         # Provider 接口定义
│   └── litellm_provider.py  # 统一 Provider 实现
├── channels/           # 通信渠道集成
│   ├── base.py         # 渠道基类
│   ├── telegram.py     # Telegram Bot 集成
│   ├── whatsapp.py     # WhatsApp 集成（通过 Node.js bridge）
│   └── feishu.py       # 飞书/Lark 集成（WebSocket 长连接）
├── bus/                # 消息总线
│   ├── events.py       # 事件类型定义
│   └── queue.py       # 异步消息队列
├── session/             # 会话管理
│   └── manager.py     # Session 持久化
├── config/              # 配置系统
│   ├── schema.py       # Pydantic 配置模型
│   └── loader.py       # 配置加载
├── cron/               # 调度系统
│   ├── service.py      # Cron 服务
│   └── types.py       # Cron 类型定义
├── heartbeat/           # 心跳服务
│   └── service.py     # 定时唤醒
├── skills/             # 内置技能
│   ├── github/        # GitHub 集成
│   ├── weather/       # 天气查询
│   ├── tmux/          # Tmux 管理
│   ├── cron/          # 调度任务
│   ├── skill-creator/ # 技能创建器
│   └── summarize/     # 内容摘要
└── cli/                # 命令行接口
    └── commands.py     # Typer 命令定义
```

### 🔑 关键设计原则
1. **最小化依赖**: 使用标准库（asyncio, pathlib, dataclasses）
2. **异步优先**: 全架构基于 asyncio
3. **可扩展性**: 技能系统和 Provider 可插拔
4. **类型安全**: 使用 Pydantic 和类型注解
5. **配置驱动**: 所有行为通过 JSON 配置控制

---

## 核心架构模式

### 🔄 Agent 循环模式

Nanobot 实现了经典的 **ReAct (Reason + Act)** 模式，这是现代 AI Agent 的核心架构。

#### 工作流程

```python
# 1. 接收消息
msg = await bus.consume_inbound()

# 2. 构建上下文
messages = context.build_messages(
    history=session.get_history(),
    current_message=msg.content,
    media=msg.media,
    channel=msg.channel,
    chat_id=msg.chat_id
)

# 3. Agent 循环
while iteration < max_iterations:
    # 3a. 调用 LLM
    response = await provider.chat(
        messages=messages,
        tools=tools.get_definitions(),
        model=model
    )

    # 3b. 处理工具调用
    if response.has_tool_calls:
        for tool_call in response.tool_calls:
            result = await tools.execute(tool_call.name, tool_call.arguments)
            messages = context.add_tool_result(messages, tool_call.id, tool_call.name, result)

    # 3c. 判断是否完成
    else:
        final_content = response.content
        break

# 4. 保存并响应
session.add_message("user", msg.content)
session.add_message("assistant", final_content)
await bus.publish_outbound(OutboundMessage(...))
```

#### 关键特性

**1. 工具调用闭环**
- LLM 返回工具调用列表
- Agent 依次执行每个工具
- 将执行结果回传给 LLM
- LLM 基于结果决定下一步操作
- 最多迭代 20 次防止无限循环

**2. 多轮对话管理**
- Session 存储历史消息
- 每次调用 LLM 时传入历史上下文
- 支持滚动窗口（最近 N 条消息）

**3. 系统消息支持**
- 子 Agent 通过 "system" 频道发布结果
- 主 Agent 通过 `chat_id` 字段路由回原始渠道
- 格式: `"channel:chat_id"` 用于标识来源

#### 架构优势

| 特性 | 说明 | 代码位置 |
|------|------|---------|
| **解耦合** | Bus + Provider 模式，Agent 不关心消息来源 | `loop.py` |
| **可测试** | 可以注入模拟 Provider 进行单元测试 | `loop.py:37-71` |
| **可扩展** | 新工具只需实现 Tool 接口 | `tools/base.py` |
| **状态隔离** | Session 隔离不同对话的上下文 | `session/manager.py` |

### 🧩 Provider 抽象层

#### 设计模式：抽象工厂 + 统一实现

```python
# 抽象接口（base.py）
class LLMProvider(ABC):
    @abstractmethod
    async def chat(self, messages, tools, model) -> LLMResponse:
        """统一调用接口"""
        pass

# 统一实现
class LiteLLMProvider(LLMProvider):
    def __init__(self, api_key, api_base, default_model):
        # 使用 litellm 库支持多 Provider
        self.client = litellm.completion(...)

    async def chat(self, messages, tools, model):
        # 统一处理所有 Provider
        return await self.client.acompletion(...)
```

#### 支持的 Provider

| Provider | 说明 | 特殊处理 |
|----------|------|----------|
| OpenRouter | 聚合多个 Provider，需要 `api_base` |
| Anthropic | 原生 Provider，最高优先级 |
| OpenAI | GPT 模型 |
| DeepSeek | 国内可用 Provider |
| Groq | 支持 Whisper 语音转录 |
| Gemini | Google 模型 |
| vLLM | 本地模型（需要 `api_base`） |
| Bedrock | AWS Bedrock（需特殊前缀处理） |

#### 配置优先级

```python
# schema.py:get_api_key()
return (
    self.providers.openrouter.api_key or      # 1. OpenRouter（聚合）
    self.providers.deepseek.api_key or        # 2. DeepSeek（国内）
    self.providers.anthropic.api_key or        # 3. Anthropic
    self.providers.openai.api_key or            # 4. OpenAI
    self.providers.gemini.api_key or             # 5. Gemini
    self.providers.zhipu.ai_key or              # 6. 智谱 AI
    self.providers.groq.api_key or               # 7. Groq（语音）
    self.providers.vllm.api_key or                # 8. 本地 vLLM
    None
)
```

---

## 模块深度分析

### Agent 循环系统

#### 核心类：`AgentLoop` (`agent/loop.py`)

**职责**：
1. 消息路由（用户消息 vs 系统消息）
2. 上下文构建（系统提示词 + 历史 + 记忆 + 技能）
3. LLM 调用与工具执行循环
4. 会话持久化
5. 响应发布

**关键方法**：

| 方法 | 行为 |
|------|------|
| `run()` | 主循环：持续监听消息队列 |
| `_process_message()` | 处理单条消息的完整流程 |
| `_process_system_message()` | 处理子 Agent 的系统消息 |
| `process_direct()` | 直接处理（CLI/Cron）模式 |
| `_register_default_tools()` | 注册内置工具集 |

#### 上下文构建：`ContextBuilder` (`agent/context.py`)

**职责**：
1. 组装系统提示词（身份 + 时间 + 工作空间）
2. 加载引导文件（`AGENTS.md`, `SOUL.md`, `USER.md`, `TOOLS.md`, `IDENTITY.md`）
3. 加载长期记忆（`MEMORY.md`）
4. 加载今日笔记（`memory/YYYY-MM-DD.md`）
5. 加载技能描述（progressive loading）

**Bootstrap 机制**：
```python
BOOTSTRAP_FILES = [
    "AGENTS.md",   # Agent 行为指南
    "SOUL.md",     # Agent 个性
    "USER.md",     # 用户偏好
    "TOOLS.md",     # 工具说明
    "IDENTITY.md"   # 身份配置
]
```

**渐进式技能加载**：
- `always=true` 的技能：完整内容加载到系统提示词
- 其他技能：仅显示 XML 摘要，Agent 按需使用 `read_file` 读取

### Tool 系统

#### 抽象基类：`Tool` (`agent/tools/base.py`)

**接口定义**：

```python
class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str:
        """工具名称（用于函数调用）"""

    @property
    @abstractmethod
    def description(self) -> str:
        """工具描述（LLM 用于选择）"""

    @property
    @abstractmethod
    def parameters(self) -> dict[str, Any]:
        """JSON Schema（参数验证）"""

    @abstractmethod
    async def execute(self, **kwargs) -> str:
        """执行工具并返回结果（字符串格式）"""

    def validate_params(self, params: dict) -> list[str]:
        """参数验证，返回错误列表"""
```

#### 注册表：`ToolRegistry` (`agent/tools/registry.py`)

**职责**：
1. 动态注册/注销工具
2. 批量获取 OpenAI Function Calling 格式定义
3. 统一执行接口（带错误处理）

**关键设计**：
```python
# 转换为 OpenAI 格式
def get_definitions(self) -> list[dict]:
    return [tool.to_schema() for tool in self._tools.values()]

# 统一执行
async def execute(self, name: str, params: dict) -> str:
    tool = self._tools.get(name)
    if not tool:
        return f"Error: Tool '{name}' not found"

    # 参数验证
    errors = tool.validate_params(params)
    if errors:
        return f"Error: Invalid parameters: " + "; ".join(errors)

    return await tool.execute(**params)
```

#### 内置工具集

| 工具 | 文件 | 功能 | 安全特性 |
|--------|------|------|---------|
| `ReadFileTool` | `filesystem.py` | 读取文件内容 |
| `WriteFileTool` | `filesystem.py` | 写入文件 |
| `EditFileTool` | `filesystem.py` | 编辑文件（行替换） |
| `ListDirTool` | `filesystem.py` | 列出目录 |
| `ExecTool` | `shell.py` | 执行 Shell 命令 |
| `WebSearchTool` | `web.py` | Brave 搜索 |
| `WebFetchTool` | `web.py` | 获取网页内容 |
| `MessageTool` | `message.py` | 发送消息到特定渠道 |
| `SpawnTool` | `spawn.py` | 创建子 Agent |
| `CronTool` | `cron.py` | 管理调度任务 |

**Shell 执行安全**：
```python
class ExecToolConfig(BaseModel):
    timeout: int = 60                    # 超时保护
    restrict_to_workspace: bool = False  # 工作空间限制

class ExecTool(Tool):
    async def execute(self, command: str) -> str:
        # 工作目录限制检查
        if self.config.restrict_to_workspace:
            if not Path(command).resolve().is_relative_to(self.working_dir):
                return "Error: Command would access path outside workspace"
```

### Session 管理

#### 类：`SessionManager` (`session/manager.py`)

**存储格式**：JSONL（每行一个 JSON 对象）

```
{"_type": "metadata", "created_at": "2026-02-05T...", "updated_at": "...", "metadata": {...}}
{"role": "user", "content": "...", "timestamp": "..."}
{"role": "assistant", "content": "...", "timestamp": "..."}
```

**缓存机制**：
```python
# 内存缓存（避免重复磁盘 I/O）
self._cache: dict[str, Session] = {}

def get_or_create(self, key: str) -> Session:
    # 1. 检查缓存
    if key in self._cache:
        return self._cache[key]

    # 2. 尝试加载磁盘
    session = self._load(key)
    if session is None:
        session = Session(key=key)

    # 3. 更新缓存
    self._cache[key] = session
    return session
```

**历史管理**：
```python
def get_history(self, max_messages: int = 50) -> list:
    # 获取最近 N 条消息
    recent = self.messages[-max_messages:]

    # 转换为 LLM 格式（仅 role + content）
    return [{"role": m["role"], "content": m["content"]} for m in recent]
```

### Memory 系统

#### 类：`MemoryStore` (`agent/memory.py`)

**双重记忆结构**：

1. **长期记忆**：`memory/MEMORY.md`
   - 用户重要信息
   - 偏好设置
   - 需要记住的事项

2. **每日笔记**：`memory/YYYY-MM-DD.md`
   - 当天的临时记录
   - 自动按日期组织

**查询 API**：
```python
def get_memory_context(self) -> str:
    parts = []

    # 长期记忆
    long_term = self.read_long_term()
    if long_term:
        parts.append("## Long-term Memory\n" + long_term)

    # 今日笔记
    today = self.read_today()
    if today:
        parts.append("## Today's Notes\n" + today)

    return "\n\n".join(parts) if parts else ""
```

**查询范围**：
```python
def get_recent_memories(self, days: int = 7) -> str:
    # 查询最近 N 天的记忆
    for i in range(days):
        date = today - timedelta(days=i)
        date_str = date.strftime("%Y-%m-%d")
        file_path = self.memory_dir / f"{date_str}.md"
        if file_path.exists():
            content = file_path.read_text()
            memories.append(content)
```

### Skills 系统

#### 类：`SkillsLoader` (`agent/skills.py`)

**技能优先级**：
1. **Workspace 技能**（用户自定义）：最高优先级
2. **Built-in 技能**：内置技能集

**技能元数据**（YAML frontmatter）：
```yaml
---
description: "技能描述"
requires:
  bins: ["git", "node"]      # 需要的二进制工具
  env: ["API_KEY"]           # 需要的环境变量
always: false                    # 是否始终加载（progressive loading）
available: true               # 是否可用（依赖检查通过）
---
```

**依赖检查**：
```python
def _check_requirements(self, skill_meta: dict) -> bool:
    requires = skill_meta.get("requires", {})

    # 检查二进制工具
    for b in requires.get("bins", []):
        if not shutil.which(b):
            return False

    # 检查环境变量
    for env in requires.get("env", []):
        if not os.environ.get(env):
            return False

    return True
```

**Progressive Loading 模式**：
```python
# 系统提示词中包含：
# 1. Always 技能（完整内容）
always_skills = self.skills.get_always_skills()
always_content = self.skills.load_skills_for_context(always_skills)
parts.append(f"# Active Skills\n\n{always_content}")

# 2. 其他技能（仅摘要）
skills_summary = self.skills.build_skills_summary()
# 格式：XML <skills><skill available="true">...</skill></skills>
parts.append(f"""# Skills
The following skills extend your capabilities. To use a skill, read its SKILL.md file using read_file tool.
{skills_summary}""")
```

### Subagent 管理

#### 类：`SubagentManager` (`agent/subagent.py`)

**设计目标**：将复杂任务委托给专用子 Agent，主 Agent 继续处理用户交互。

**子 Agent 特征**：
1. **隔离上下文**：不访问主 Agent 的会话历史
2. **限制工具集**：只允许文件、Shell、Web 工具（无 message、spawn、cron）
3. **专注提示词**：明确的任务导向系统提示词
4. **结果通知**：通过系统消息返回结果

**生命周期**：
```python
# 1. 生成任务 ID
task_id = str(uuid.uuid4())[:8]

# 2. 创建异步任务
bg_task = asyncio.create_task(
    self._run_subagent(task_id, task, display_label, origin)
)
self._running_tasks[task_id] = bg_task

# 3. 完成时清理
bg_task.add_done_callback(lambda _: self._running_tasks.pop(task_id, None))
```

**系统消息格式**：
```python
# 主 Agent 解析并路由回原始渠道
if msg.channel == "system":
    if ":" in msg.chat_id:
        origin_channel = msg.chat_id.split(":", 1)[0]
        origin_chat_id = msg.chat_id.split(":", 1)[1]

# 子 Agent 发布结果
msg = InboundMessage(
    channel="system",
    sender_id="subagent",
    chat_id=f"{origin_channel}:{origin_chat_id}",
    content=f"[Subagent '{label}' {status_text}]\n\nTask: {task}\n\nResult:\n{result}"
)
await bus.publish_inbound(msg)
```

### Communication Channels

#### 基类：`BaseChannel` (`channels/base.py`)

**统一接口**：
```python
class BaseChannel(ABC):
    name: str                          # 渠道标识
    config: Any                       # 渠道配置

    @abstractmethod
    async def start(self) -> None:    # 启动连接
    @abstractmethod
    async def stop(self) -> None:     # 停止连接
    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None:  # 发送消息
```

#### Telegram 实现 (`channels/telegram.py`)

**特点**：
- 使用 `python-telegram-bot` 库
- Long polling 模式（无需 webhook/公网 IP）
- 支持 Markdown → HTML 转换
- 语音转录（通过 Groq API）
- 文件下载和媒体处理

**Markdown 转换**：
```python
def _markdown_to_telegram_html(text: str) -> str:
    # 保护代码块（使用 Unicode 零宽字符）
    text = re.sub(r'```[\w]*\n?([\s\S]*?)```',
                  save_code_block, text)

    # 保护行内代码
    text = re.sub(r'`([^`]+)`', save_inline_code, text)

    # 格式转换
    # **text** → <b>text</b>
    # ~~text~~ → <s>text</s>
    # [text](url) → <a href="url">text</a>
```

**语音转录流程**：
```python
if media_type == "voice" or media_type == "audio":
    transcriber = GroqTranscriptionProvider(api_key=self.groq_api_key)
    transcription = await transcriber.transcribe(file_path)
    if transcription:
        content_parts.append(f"[transcription: {transcription}]")
```

#### WhatsApp 实现 (`channels/whatsapp.py`)

**架构**：Python + Node.js Bridge
```
Python (WhatsAppChannel) <--> WebSocket <--> Node.js (@whiskeysockets/baileys)
```

**Bridge 协议**：
```json
// Python → Node.js
{"type": "send", "to": "+8613800xxxx", "text": "..."}

// Node.js → Python
{"type": "message", "sender": "+8613800xxxx@whatsapp.net", "content": "..."}
{"type": "status", "status": "connected"/"disconnected"}
{"type": "qr", ...}  // QR 码
```

#### Feishu/Lark 实现 (`channels/feishu.py`)

**特点**：
- 使用 `lark-oapi` SDK
- WebSocket 长连接模式（无需 webhook/公网 IP）
- 消息去重（`_processed_message_ids: OrderedDict`）
- 线程安全（WebSocket 在独立线程）

**去重机制**：
```python
# 使用 OrderedDict 保持插入顺序
self._processed_message_ids: OrderedDict[str, None] = OrderedDict()

# 检查并更新缓存
while len(self._processed_message_ids) > 1000:
    self._processed_message_ids.popitem(last=False)

# 跳过重复消息
if message_id in self._processed_message_ids:
    return

self._processed_message_ids[message_id] = None
```

### Message Bus

#### 类：`MessageBus` (`bus/queue.py`)

**职责**：异步消息队列和发布/订阅机制

```python
class MessageBus:
    def __init__(self):
        self._inbound_queue = asyncio.Queue()  # 入站消息
        self._outbound_subscribers = []         # 出站订阅者

    async def publish_inbound(self, msg: InboundMessage):
        """发布入站消息到队列"""
        await self._inbound_queue.put(msg)

    async def consume_inbound(self):
        """从队列消费消息（带超时）"""
        return await asyncio.wait_for(self._inbound_queue.get(), timeout=1.0)

    async def publish_outbound(self, msg: OutboundMessage):
        """通知所有出站订阅者"""
        for callback in self._outbound_subscribers:
            if asyncio.iscoroutinefunction(callback):
                await callback(msg)
            else:
                callback(msg)

    def subscribe_outbound(self, callback):
        """订阅出站消息"""
        self._outbound_subscribers.append(callback)
```

**设计优势**：
- **异步解耦**：生产者/消费者通过队列通信
- **多订阅者支持**：一个消息可通知多个接收者
- **超时控制**：避免无限阻塞（`timeout=1.0`）

### Cron 调度系统

#### 类：`CronService` (`cron/service.py`)

**调度类型**：
1. **Every**：每 N 秒执行（`every_ms: N * 1000`）
2. **Cron**：标准 cron 表达式（`expr: "0 9 * * *"`）
3. **At**：单次执行（`at_ms: timestamp`）

**存储格式**（JSON）：
```json
{
  "jobs": {
    "job_id": {
      "id": "job_id",
      "name": "job_name",
      "schedule": {
        "kind": "every"/"cron"/"at",
        "every_ms": 3600000,  // 每小时
        "expr": "0 9 * * *",
        "at_ms": 1738769200000
      },
      "payload": {
        "message": "Good morning!",
        "channel": "telegram",
        "to": "user123",
        "deliver": true  // 是否需要发送响应到渠道
      },
      "state": {
        "enabled": true,
        "next_run_at_ms": 1738772800000  // 下次执行时间（毫秒）
      }
    }
  }
}
```

**回调机制**：
```python
# Agent 设置回调
async def on_cron_job(job: CronJob) -> str:
    response = await agent.process_direct(
        job.payload.message,
        session_key=f"cron:{job.id}",
        channel=job.payload.channel or "cli",
        chat_id=job.payload.to or "direct"
    )
    if job.payload.deliver and job.payload.to:
        await bus.publish_outbound(OutboundMessage(...))

cron.on_job = on_cron_job  # 注册回调
```

### Configuration 系统

#### 配置模型：`Config` (`config/schema.py`)

**使用 Pydantic Settings**：
```python
class Config(BaseSettings):
    # 自动从环境变量加载（支持嵌套）
    env_prefix = "NANOBOT_"
    env_nested_delimiter = "__"

    agents: AgentsConfig = Field(default_factory=AgentsConfig)
    channels: ChannelsConfig = Field(default_factory=ChannelsConfig)
    providers: ProvidersConfig = Field(default_factory=ProvidersConfig)
    gateway: GatewayConfig = Field(default_factory=GatewayConfig)
    tools: ToolsConfig = Field(default_factory=ToolsConfig)
```

**环境变量映射**：
```bash
# 文件: ~/.nanobot/config.json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-4-5"
    }
  },
  "providers": {
    "anthropic": {
      "apiKey": "sk-ant-..."
    }
  }
}

# 等价的环境变量：
export NANOBOT_AGENTS__DEFAULTS__MODEL="anthropic/claude-opus-4-5"
export NANOBOT_PROVIDERS__ANTHROPIC__APIKEY="sk-ant-..."
```

#### 配置加载器：`load_config()` (`config/loader.py`)

**优先级**：环境变量 > 配置文件 > 默认值

```python
# Pydantic Settings 自动加载环境变量
config = Config()

# 如果文件不存在，使用默认值
config_path = get_config_path()
if not config_path.exists():
    config = Config()  # 使用默认值
```

### CLI 架构

#### 框架：Typer (`cli/commands.py`)

**命令结构**：
```python
app = typer.Typer(name="nanobot", help="...")

# 主要命令组
@app.command()
def onboard():      # 初始化配置
@app.command()
def agent():        # 直接对话
@app.command()
def gateway():      # 启动网关（所有渠道）
@app.command()
def status():       # 显示状态

# 子命令组
channels_app = typer.Typer(help="Manage channels")
@channels_app.command("status")
def channels_status():  # 显示渠道状态
@channels_app.command("login")
def channels_login():   # WhatsApp 登录

cron_app = typer.Typer(help="Manage scheduled tasks")
@cron_app.command("list")
def cron_list():     # 列出任务
@cron_app.command("add")
def cron_add():      # 添加任务
@cron_app.command("remove")
def cron_remove():   # 删除任务
@cron_app.command("run")
def cron_run():      # 手动运行
@cron_app.command("enable")
def cron_enable():   # 启用任务
```

**交互模式**：
```python
# 单消息模式
nanobot agent -m "Hello"

# 交互模式
nanobot agent
# 循环读取输入并处理
while True:
    user_input = console.input("[bold blue]You:[/bold blue] ")
    response = await agent_loop.process_direct(user_input)
    console.print(f"\n{response}\n")
```

---

## 设计决策与最佳实践

### 🎯 核心设计原则

| 原则 | 实现 | 原因 |
|------|------|--------|
| **最小化依赖** | 仅使用 asyncio, pathlib, dataclasses, pydantic | 减小安装体积，提高可移植性 |
| **异步优先** | 全架构基于 async/await | 避免阻塞，支持并发处理 |
| **接口隔离** | Provider/Tool/Channel 全部使用抽象基类 | 易于测试和替换 |
| **配置驱动** | 所有行为通过 JSON 配置 | 无需重新编译即可调整 |
| **渐进式加载** | Skills 和 Memory 按需加载 | 减少上下文大小，提高响应速度 |
| **错误容错** | 统一的 try-except 包装 | 提供清晰错误信息 |
| **类型安全** | 全面使用类型注解和 Pydantic | 静态类型检查，IDE 支持 |
| **日志友好** | 使用 Loguru 结构化日志 | 便于调试和监控 |

### 📦 数据流设计

#### 消息流

```
用户 → [Channel] → [MessageBus] → [AgentLoop] → [Provider] → [Tool Execution]
     ↓
[Subagent] ↗
     ↓
[AgentLoop] → [Outbound] → [Channel] → 用户
```

#### 状态管理流

```
Config (Pydantic) → 优先级: 环境变量 > 文件 > 默认
                           ↓
        [Session] → JSONL 持久化 → 内存缓存
                           ↓
        [Memory] → 文件系统组织 → 渐进式查询
```

### 🔒 安全特性

#### Shell 执行限制
```python
class ExecToolConfig:
    restrict_to_workspace: bool = False  # 默认不限制
    timeout: int = 60                # 默认超时 60 秒

class ExecTool:
    async def execute(self, command: str) -> str:
        if self.config.restrict_to_workspace:
            path = Path(command).resolve()
            if not path.is_relative_to(self.working_dir):
                return "Error: Command would access path outside workspace"
```

#### 参数验证
```python
def validate_params(self, params: dict) -> list[str]:
    errors = []

    # 类型检查
    schema = self.parameters or {}
    if schema.get("type") != "object":
        return ["Schema must be object type"]

    # 必需字段检查
    for k in schema.get("required", []):
        if k not in val:
            errors.append(f"missing required field: {k}")

    # 枚举值检查
    if "enum" in schema and val not in schema["enum"]:
        errors.append(f"must be one of: {schema['enum']}")

    # 范围检查
    if "minimum" in schema and val < schema["minimum"]:
        errors.append(f"must be >= {schema['minimum']}")

    return errors  # 返回所有错误（验证失败时）
```

#### 消息去重
```python
# Feishu WebSocket 使用 OrderedDict
self._processed_message_ids: OrderedDict[str, None] = OrderedDict()

# 限制缓存大小（保留最近 1000）
while len(self._processed_message_ids) > 1000:
    self._processed_message_ids.popitem(last=False)

# 检查并跳过重复
if message_id in self._processed_message_ids:
    return  # 不处理
```

### 🚀 性能优化

| 优化技术 | 实现 | 效果 |
|---------|------|------|
| **内存缓存** | Session/Memory 的 `_cache` 字典 | 避免重复磁盘 I/O |
| **渐进式加载** | Skills 仅显示摘要，按需加载 | 减少提示词 Token 消耗 |
| **异步并发** | `asyncio.gather()` 启动多个服务 | 同时运行 Agent + Channels + Cron |
| **滚动窗口** | Session.get_history(max_messages=50) | 限制上下文大小，提高速度 |
| **批量保存** | Cron 一次性保存所有任务 | 减少磁盘写入次数 |

---

## 学习路径

### 📚 推荐学习顺序

#### 阶段 1：基础架构理解（1-2 周）
1. 阅读 `agent/loop.py` 理解 ReAct 模式
2. 阅读 `agent/context.py` 理解 Prompt 构建
3. 阅读 `bus/queue.py` 理解异步队列
4. 阅读 `agent/tools/base.py` 理解 Tool 接口

**实践**：
- 修改 Loop 的最大迭代次数，观察行为变化
- 添加一个新的简单工具（如 `echo`）
- 调整 Context 的系统提示词，观察 LLM 响应

#### 阶段 2：可扩展性设计（2-3 周）
1. 研究 `providers/base.py` 和 `litellm_provider.py`
2. 创建一个新的 Provider 实现（如 Ollama）
3. 学习 Skills 系统的元数据机制

**实践**：
- 添加新的 Provider 支持
- 创建一个自定义 Skill（如 `translate`）
- 修改 Skill 的依赖检查逻辑

#### 阶段 3：集成与部署（4-5 周）
1. 研究 `channels/telegram.py` 理解 Bot 集成
2. 研究 `channels/feishu.py` 理解 WebSocket 长连接
3. 研究 `bridge/` 目录理解 Node.js 集成
4. 研究 `cron/` 理解调度系统

**实践**：
- 集成一个新的 Channel（如 Slack）
- 修改 Cron 表达式，测试复杂调度
- 部署到服务器并配置网关

#### 阶段 4：高级特性（5-6 周）
1. 研究 Subagent 管理的隔离机制
2. 研究内存系统的渐进式查询
3. 研究错误处理和重试逻辑
4. 学习 Pydantic 配置管理的最佳实践

**实践**：
- 实现工具参数的复杂验证
- 添加内存的向量搜索（可选）
- 实现子 Agent 的并行执行
- 添加健康检查端点

#### 阶段 5：生产化与优化（7-8 周）
1. 性能分析和优化
2. 监控和日志
3. 测试覆盖率提升
4. 文档生成

**实践**：
- 添加性能指标收集
- 实现日志轮转
- 添加单元测试和集成测试
- 生成 API 文档

### 📖 关键技术概念

| 概念 | 说明 | 代码位置 |
|------|------|---------|
| **ReAct 模式** | Reason + Act 循环 | `loop.py:182-224` |
| **Function Calling** | OpenAI 格式的工具调用 | `tools/base.py:93-102` |
| **Progressive Loading** | 技能按需加载策略 | `context.py:52-68` |
| **Abstract Factory** | Provider 抽象层设计 | `providers/base.py:30-70` |
| **Pub/Sub 模式** | 消息总线架构 | `bus/queue.py` |
| **JSONL 格式** | Session 持久化格式 | `session/manager.py:203` |
| **Async Context** | asyncio.run_coroutine_threadsafe() | `channels/feishu.py:204-205` |
| **Pydantic Settings** | 配置自动加载 | `config/schema.py:38-41` |

### 🔧 代码质量指标

| 指标 | 当前值 | 目标 | 改进方向 |
|------|--------|------|---------|
| **代码行数** | ~5,302 行 Python | 保持精简 |
| **平均文件长度** | ~120 行 | 文件职责单一 |
| **类型覆盖率** | 大部分有类型注解 | 100% 覆盖 |
| **文档率** | 所有类和公共方法有 docstring | 完善 docstring |
| **测试** | 无测试文件 | 添加单元测试 |
| **日志** | 使用 Loguru | 添加性能指标 |

---

## 附录：核心文件索引

| 文件 | 行数 | 核心功能 |
|------|------|---------|
| `agent/loop.py` | 366 | Agent 主循环 |
| `agent/context.py` | 224 | Prompt 构建 |
| `agent/memory.py` | 110 | 记忆管理 |
| `agent/skills.py` | 229 | 技能加载器 |
| `agent/subagent.py` | 242 | 子 Agent 管理 |
| `agent/tools/base.py` | 103 | 工具基类 |
| `agent/tools/registry.py` | 74 | 工具注册表 |
| `providers/base.py` | 70 | Provider 接口 |
| `session/manager.py` | 203 | Session 管理 |
| `channels/telegram.py` | 303 | Telegram 集成 |
| `channels/whatsapp.py` | 142 | WhatsApp 集成 |
| `channels/feishu.py` | 264 | Feishu 集成 |
| `bus/queue.py` | 未知 | 消息总线 |
| `config/schema.py` | 141 | 配置模型 |
| `cli/commands.py` | 661 | CLI 命令 |

---

## 总结

Nanobot 是一个设计精良的轻量级 AI Agent 框架，展示了：

1. **清晰的架构分层**：Agent → Provider → Tool → Channel → Bus
2. **异步优先设计**：全栈异步，避免阻塞
3. **可扩展机制**：Skills, Providers, Channels 全部可插拔
4. **渐进式加载**：优化 Token 使用，提高响应速度
5. **配置驱动**：无需重新编译即可调整行为
6. **安全特性**：Shell 限制、参数验证、去重机制

**建议学习重点**：
1. 理解 ReAct 模式的实现细节
2. 学习如何设计可扩展的 Tool 系统
3. 掌握异步消息总线的设计模式
4. 研究多 Provider 的抽象和统一实现
5. 理解 Session 和 Memory 的持久化策略

---

*报告生成工具：AI 自动分析*
*报告版本：v1.0*
