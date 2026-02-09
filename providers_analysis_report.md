# nanobot providers 目录深度分析报告

> 分析日期: 2026年2月9日
> 项目版本: v0.1.3.post4
> 目标: 学习 nanobot\providers 目录的架构设计和编程知识

---

## 目录

1. [概述](#概述)
2. [目录结构](#目录结构)
3. [核心架构设计](#核心架构设计)
4. [详细实现分析](#详细实现分析)
5. [集成模式](#集成模式)
6. [设计模式总结](#设计模式总结)
7. [学习要点](#学习要点)

---

## 概述

`nanobot/providers` 目录是整个 AI 助手系统的 **核心抽象层**，负责统一不同 LLM 提供商的 API 调用。该目录实现了一个灵活、可扩展的提供者系统，支持多种 LLM 服务（OpenRouter、Anthropic、OpenAI、Gemini、DeepSeek、Groq、vLLM 等）以及语音转录服务。

### 关键功能

- **统一接口**: 为所有 LLM 提供商提供统一的调用接口
- **工具调用**: 支持 Function Calling（工具调用）机制
- **多提供商支持**: 通过 LiteLLM 库实现多提供商统一调用
- **语音转文字**: 集成 Groq Whisper API 实现语音转录

---

## 目录结构

```
nanobot/
└── providers/
    ├── __init__.py           # 模块导出
    ├── base.py               # 抽象基类定义
    ├── litellm_provider.py   # LLM 提供商实现（多提供商支持）
    └── transcription.py      # 语音转录提供商实现
```

### 文件功能概览

| 文件 | 代码行数 | 职责 |
|------|---------|------|
| `__init__.py` | 7 | 模块导出接口 |
| `base.py` | 70 | 定义抽象基类和数据结构 |
| `litellm_provider.py` | 176 | LLM 提供商具体实现 |
| `transcription.py` | 66 | 语音转录服务实现 |

**总计**: ~319 行核心代码

---

## 核心架构设计

### 1. 抽象基类模式 (Abstract Base Pattern)

providers 使用 Python 的 `abc.ABC`（抽象基类）定义接口契约：

```python
from abc import ABC, abstractmethod

class LLMProvider(ABC):
    """
    LLM 提供商抽象基类

    实现类必须处理各提供商的 API 细节，
    同时保持接口一致性。
    """

    def __init__(self, api_key: str | None = None, api_base: str | None = None):
        self.api_key = api_key
        self.api_base = api_base

    @abstractmethod
    async def chat(
        self,
        messages: list[dict[str, Any]],
        tools: list[dict[str, Any]] | None = None,
        model: str | None = None,
        max_tokens: int = 4096,
        temperature: float = 0.7,
    ) -> LLMResponse:
        """
        发送聊天完成请求

        Args:
            messages: 消息字典列表，包含 'role' 和 'content'
            tools: 可选的工具定义列表
            model: 模型标识符（提供商特定）
            max_tokens: 响应最大 token 数
            temperature: 采样温度

        Returns:
            包含内容和/或工具调用的 LLMResponse
        """
        pass

    @abstractmethod
    def get_default_model(self) -> str:
        """获取此提供商的默认模型。"""
        pass
```

**设计要点**:
- 使用 `@abstractmethod` 强制子类实现核心方法
- 构造函数接受通用参数（api_key, api_base）
- 类型注解（Python 3.11+ 的 `str | None` 语法）提供类型安全
- 文档字符串（docstring）详细说明每个方法的用途

### 2. 数据类模式 (Dataclass Pattern)

使用 Python `dataclasses` 定义数据结构，减少样板代码：

```python
@dataclass
class ToolCallRequest:
    """来自 LLM 的工具调用请求"""
    id: str                    # 工具调用 ID（用于关联响应）
    name: str                  # 工具名称
    arguments: dict[str, Any]   # 工具参数（JSON 字典）

@dataclass
class LLMResponse:
    """来自 LLM 提供商的响应"""
    content: str | None                        # 响应文本内容
    tool_calls: list[ToolCallRequest] = field(default_factory=list)  # 工具调用列表
    finish_reason: str = "stop"               # 结束原因（stop/tool_calls/error）
    usage: dict[str, int] = field(default_factory=dict)  # Token 使用统计

    @property
    def has_tool_calls(self) -> bool:
        """检查响应是否包含工具调用"""
        return len(self.tool_calls) > 0
```

**设计要点**:
- `field(default_factory=list)` 用于可变默认值（避免共享引用问题）
- 使用 `@property` 装饰器添加便利方法
- 数据类自动生成 `__init__`, `__repr__`, `__eq__` 等方法

---

## 详细实现分析

### 3.1 LiteLLMProvider - 多提供商实现

#### 3.1.1 架构概述

`LiteLLMProvider` 通过包装 [litellm](https://docs.litellm.ai/) 库实现多提供商支持。LiteLLM 是一个统一接口库，可以将不同 LLM 提供商的 API 标准化为 OpenAI 格式。

```python
class LiteLLMProvider(LLMProvider):
    """
    使用 LiteLLM 实现多提供商支持的 LLM 提供商

    通过统一接口支持 OpenRouter、Anthropic、OpenAI、Gemini
    以及许多其他提供商。
    """
```

#### 3.1.2 初始化逻辑

```python
def __init__(
    self,
    api_key: str | None = None,
    api_base: str | None = None,
    default_model: str = "anthropic/claude-opus-4-5"
):
    super().__init__(api_key, api_base)
    self.default_model = default_model

    # 通过 api_key 前缀或明确的 api_base 检测 OpenRouter
    self.is_openrouter = (
        (api_key and api_key.startswith("sk-or-")) or
        (api_base and "openrouter" in api_base)
    )

    # 跟踪是否使用自定义端点（vLLM 等）
    self.is_vllm = bool(api_base) and not self.is_openrouter

    # 根据提供商配置 LiteLLM
    if api_key:
        if self.is_openrouter:
            os.environ["OPENROUTER_API_KEY"] = api_key
        elif self.is_vllm:
            os.environ["OPENAI_API_KEY"] = api_key
        elif "deepseek" in default_model:
            os.environ.setdefault("DEEPSEEK_API_KEY", api_key)
        elif "anthropic" in default_model:
            os.environ.setdefault("ANTHROPIC_API_KEY", api_key)
        elif "openai" in default_model or "gpt" in default_model:
            os.environ.setdefault("OPENAI_API_KEY", api_key)
        elif "gemini" in default_model.lower():
            os.environ.setdefault("GEMINI_API_KEY", api_key)
        elif "zhipu" in default_model or "glm" in default_model or "zai" in default_model:
            os.environ.setdefault("ZHIPUAI_API_KEY", api_key)
        elif "groq" in default_model:
            os.environ.setdefault("GROQ_API_KEY", api_key

    if api_base:
        litellm.api_base = api_base

    # 禁用 LiteLLM 日志噪音
    litellm.suppress_debug_info = True
```

**关键设计决策**:

1. **自动提供商检测**: 通过 API key 前缀或 model 名称自动识别提供商
   - `sk-or-*` → OpenRouter
   - 包含 "deepseek" → DeepSeek
   - 包含 "anthropic" → Anthropic
   - 等等...

2. **环境变量配置**: 使用 `os.environ.setdefault()` 设置环境变量
   - 使用 `setdefault` 而非直接赋值，避免覆盖用户已设置的环境变量
   - LiteLLM 库从环境变量读取 API 密钥

3. **本地模型支持**: vLLM 模式使用 OpenAI 兼容 API
   - 设置 `api_base` 到本地服务器地址
   - 使用 `hosted_vllm/` 前缀

#### 3.1.3 模型名称规范化

不同提供商的模型名称需要添加特定前缀：

```python
async def chat(self, ...):
    model = model or self.default_model

    # OpenRouter 模型需要前缀
    if self.is_openrouter and not model.startswith("openrouter/"):
        model = f"openrouter/{model}"

    # Zhipu/Z.ai 确保有前缀
    if ("glm" in model.lower() or "zhipu" in model.lower()) and not (
        model.startswith("zhipu/") or
        model.startswith("zai/") or
        model.startswith("openrouter/")
    ):
        model = f"zai/{model}"

    # vLLM 使用 hosted_vllm/ 前缀
    if self.is_vllm:
        model = f"hosted_vllm/{model}"

    # Gemini 确保 gemini/ 前缀
    if "gemini" in model.lower() and not model.startswith("gemini/"):
        model = f"gemini/{model}"
```

**为什么需要前缀**:
- LiteLLM 使用前缀来区分不同的提供商
- 例如: `anthropic/claude-3-opus` vs `openrouter/anthropic/claude-3-opus`

#### 3.1.4 工具调用支持

```python
kwargs: dict[str, Any] = {
    "model": model,
    "messages": messages,
    "max_tokens": max_tokens,
    "temperature": temperature,
}

if tools:
    kwargs["tools"] = tools
    kwargs["tool_choice"] = "auto"  # 让 LLM 自动决定是否调用工具
```

**工具调用流程**:
1. 将工具定义列表传递给 LLM（OpenAI Function Calling 格式）
2. LLM 返回时可能包含 `tool_calls` 字段
3. 解析工具调用并执行相应操作

#### 3.1.5 响应解析

```python
def _parse_response(self, response: Any) -> LLMResponse:
    """将 LiteLLM 响应解析为我们的标准格式"""
    choice = response.choices[0]
    message = choice.message

    tool_calls = []
    if hasattr(message, "tool_calls") and message.tool_calls:
        for tc in message.tool_calls:
            # 从 JSON 字符串解析参数（如果需要）
            args = tc.function.arguments
            if isinstance(args, str):
                try:
                    args = json.loads(args)
                except json.JSONDecodeError:
                    args = {"raw": args}

            tool_calls.append(ToolCallRequest(
                id=tc.id,
                name=tc.function.name,
                arguments=args,
            ))

    usage = {}
    if hasattr(response, "usage") and response.usage:
        usage = {
            "prompt_tokens": response.usage.prompt_tokens,
            "completion_tokens": response.usage.completion_tokens,
            "total_tokens": response.usage.total_tokens,
        }

    return LLMResponse(
        content=message.content,
        tool_calls=tool_calls,
        finish_reason=choice.finish_reason or "stop",
        usage=usage,
    )
```

**关键点**:
- 使用 `hasattr()` 检查属性是否存在（兼容不同响应格式）
- 工具参数可能是字符串（JSON）或字典，需要处理两种情况
- 提取 token 使用统计（用于计费和监控）

#### 3.1.6 错误处理

```python
try:
    response = await acompletion(**kwargs)
    return self._parse_response(response)
except Exception as e:
    # 返回错误作为内容以优雅处理
    return LLMResponse(
        content=f"Error calling LLM: {str(e)}",
        finish_reason="error",
    )
```

**设计决策**: 将错误作为 LLMResponse.content 返回，而不是抛出异常
- 允许上层调用者继续处理
- 用户可以看到错误消息
- 不会导致整个流程崩溃

### 3.2 GroqTranscriptionProvider - 语音转录

#### 3.2.1 架构概述

```python
class GroqTranscriptionProvider:
    """
    使用 Groq 的 Whisper API 的语音转录提供商

    Groq 提供极快的转录速度和慷慨的免费层
    """
```

**注意**: 这个类**不继承** `LLMProvider`，因为它是专门的语音转文字服务，不是 LLM。

#### 3.2.2 初始化

```python
def __init__(self, api_key: str | None = None):
    self.api_key = api_key or os.environ.get("GROQ_API_KEY")
    self.api_url = "https://api.groq.com/openai/v1/audio/transcriptions"
```

- 支持直接传入 api_key 或从环境变量读取
- 使用 Groq 的 OpenAI 兼容 API 端点

#### 3.2.3 转录方法

```python
async def transcribe(self, file_path: str | Path) -> str:
    """
    使用 Groq 转录音频文件

    Args:
        file_path: 音频文件路径

    Returns:
        转录文本
    """
    if not self.api_key:
        logger.warning("未配置 Groq API 密钥用于转录")
        return ""

    path = Path(file_path)
    if not path.exists():
        logger.error(f"音频文件未找到: {file_path}")
        return ""

    try:
        async with httpx.AsyncClient() as client:
            with open(path, "rb") as f:
                files = {
                    "file": (path.name, f),
                    "model": (None, "whisper-large-v3"),
                }
                headers = {
                    "Authorization": f"Bearer {self.api_key}",
                }

                response = await client.post(
                    self.api_url,
                    headers=headers,
                    files=files,
                    timeout=60.0
                )

                response.raise_for_status()
                data = response.json()
                return data.get("text", "")

    except Exception as e:
        logger.error(f"Groq 转录错误: {e}")
        return ""
```

**关键点**:
- 使用 `httpx.AsyncClient` 进行异步 HTTP 请求
- 文件以 multipart/form-data 格式上传
- 模型使用 `whisper-large-v3`（Groq 的 Whisper 实现）
- 超时设置为 60 秒（音频文件可能较大）
- 错误时返回空字符串（优雅降级）

#### 3.2.4 集成到 Telegram 频道

在 `nanobot/channels/telegram.py` 中使用：

```python
# 处理语音转录
if media_type == "voice" or media_type == "audio":
    from nanobot.providers.transcription import GroqTranscriptionProvider
    transcriber = GroqTranscriptionProvider(api_key=self.groq_api_key)
    transcription = await transcriber.transcribe(file_path)
    if transcription:
        logger.info(f"转录 {media_type}: {transcription[:50]}...")
        content_parts.append(f"[transcription: {transcription}]")
    else:
        content_parts.append(f"[{media_type}: {file_path}]")
```

**工作流程**:
1. 用户在 Telegram 发送语音消息
2. 频道下载音频文件到本地
3. 调用 GroqTranscriptionProvider.transcribe()
4. 将转录文本附加到消息内容
5. LLM 处理包含转录文本的消息

---

## 集成模式

### 4.1 依赖注入 (Dependency Injection)

Provider 通过构造函数注入到需要它的组件：

```python
# nanobot/cli/commands.py
def gateway(...):
    # 创建 provider
    provider = LiteLLMProvider(
        api_key=config.get_api_key(),
        api_base=config.get_api_base(),
        default_model=config.agents.defaults.model
    )

    # 创建 agent（注入 provider）
    agent = AgentLoop(
        bus=bus,
        provider=provider,          # 依赖注入
        workspace=config.workspace_path,
        model=config.agents.defaults.model,
        max_iterations=config.agents.defaults.max_tool_iterations,
        brave_api_key=config.tools.web.search.api_key or None,
        exec_config=config.tools.exec,
        cron_service=cron,
    )

    # 创建子代理管理器（注入 provider）
    self.subagents = SubagentManager(
        provider=provider,          # 依赖注入
        workspace=workspace,
        bus=bus,
        model=self.model,
        brave_api_key=brave_api_key,
        exec_config=self.exec_config,
    )
```

**优势**:
- 组件解耦
- 易于测试（可以 mock provider）
- 灵活切换提供商

### 4.2 Provider 在 Agent Loop 中的使用

```python
# nanobot/agent/loop.py
async def _process_message(self, msg: InboundMessage) -> OutboundMessage | None:
    # 构建消息列表
    messages = self.context.build_messages(
        history=session.get_history(),
        current_message=msg.content,
        media=msg.media if msg.media else None,
        channel=msg.channel,
        chat_id=msg.chat_id,
    )

    # Agent 循环
    iteration = 0
    final_content = None

    while iteration < self.max_iterations:
        iteration += 1

        # 调用 LLM（核心调用点）
        response = await self.provider.chat(
            messages=messages,
            tools=self.tools.get_definitions(),  # 传递工具定义
            model=self.model
        )

        # 处理工具调用
        if response.has_tool_calls:
            # 添加助手消息（包含工具调用）
            tool_call_dicts = [
                {
                    "id": tc.id,
                    "type": "function",
                    "function": {
                        "name": tc.name,
                        "arguments": json.dumps(tc.arguments)  # 必须是 JSON 字符串
                    }
                }
                for tc in response.tool_calls
            ]
            messages = self.context.add_assistant_message(
                messages, response.content, tool_call_dicts
            )

            # 执行工具
            for tool_call in response.tool_calls:
                result = await self.tools.execute(tool_call.name, tool_call.arguments)
                messages = self.context.add_tool_result(
                    messages, tool_call.id, tool_call.name, result
                )
        else:
            # 无工具调用，完成
            final_content = response.content
            break
```

**核心循环**:
1. 调用 `provider.chat()` 获取 LLM 响应
2. 检查是否有工具调用
3. 如果有工具调用，执行并返回结果
4. 将结果添加到消息历史，重新调用 LLM
5. 重复直到 LLM 不再调用工具（最终文本响应）

### 4.3 配置加载

```python
# nanobot/config/schema.py
class Config(BaseSettings):
    providers: ProvidersConfig = Field(default_factory=ProvidersConfig)

    def get_api_key(self) -> str | None:
        """
        按优先级获取 API 密钥：
        OpenRouter > DeepSeek > Anthropic > OpenAI > Gemini > Zhipu > Groq > vLLM
        """
        return (
            self.providers.openrouter.api_key or
            self.providers.deepseek.api_key or
            self.providers.anthropic.api_key or
            self.providers.openai.api_key or
            self.providers.gemini.api_key or
            self.providers.zhipu.api_key or
            self.providers.groq.api_key or
            self.providers.vllm.api_key or
            None
        )

    def get_api_base(self) -> str | None:
        """如果使用 OpenRouter、Zhipu 或 vLLM，获取 API base URL"""
        if self.providers.openrouter.api_key:
            return self.providers.openrouter.api_base or "https://openrouter.ai/api/v1"
        if self.providers.zhipu.api_key:
            return self.providers.zhipu.api_base
        if self.providers.vllm.api_base:
            return self.providers.vllm.api_base
        return None
```

**配置策略**:
- 支持多个提供商同时配置
- 按优先级选择第一个可用的 API 密钥
- 用户只需配置一个提供商即可使用

---

## 设计模式总结

### 5.1 使用的设计模式

| 模式 | 位置 | 用途 |
|------|------|------|
| **抽象工厂 (Abstract Factory)** | `LLMProvider` 基类 | 定义创建产品族的接口 |
| **策略模式 (Strategy)** | `LiteLLMProvider` 实现多提供商 | 算法族可互换 |
| **适配器模式 (Adapter)** | LiteLLM 包装不同 LLM API | 统一不同提供商接口 |
| **依赖注入 (Dependency Injection)** | AgentLoop 构造函数 | 解耦组件依赖 |
| **数据传输对象 (DTO)** | `LLMResponse`, `ToolCallRequest` | 数据容器，无行为 |
| **工厂方法 (Factory Method)** | `get_default_model()` | 子类决定默认产品 |
| **装饰器模式 (Decorator)** | `@property` `has_tool_calls` | 添加额外行为 |

### 5.2 架构原则体现

1. **开闭原则 (Open/Closed Principle)**:
   - 对扩展开放：可以添加新的 Provider 实现
   - 对修改关闭：现有代码无需修改

2. **里氏替换原则 (Liskov Substitution)**:
   - 任何 `LLMProvider` 子类都可以替换基类使用
   - `AgentLoop` 不关心具体是哪个 Provider

3. **接口隔离原则 (Interface Segregation)**:
   - `LLMProvider` 只定义必要的方法（`chat`, `get_default_model`）
   - 不同 Provider 实现最小必要接口

4. **依赖倒置原则 (Dependency Inversion)**:
   - 高层模块（AgentLoop）依赖抽象（LLMProvider）
   - 低层模块（LiteLLMProvider）实现抽象
   - 依赖通过构造函数注入

5. **单一职责原则 (Single Responsibility)**:
   - `LLMProvider` 只负责定义接口
   - `LiteLLMProvider` 只负责 LiteLLM 集成
   - `GroqTranscriptionProvider` 只负责语音转录

---

## 学习要点

### 6.1 架构设计知识

#### 6.1.1 抽象层的重要性

```
┌─────────────────────────────────────────┐
│         AgentLoop (高层)            │
│  - 不关心具体是哪个 LLM 提供商      │
└──────────────┬──────────────────────┘
               │ 依赖抽象
               ↓
┌─────────────────────────────────────────┐
│       LLMProvider (抽象接口)          │
│  + chat()                           │
│  + get_default_model()                │
└──────────────┬──────────────────────┘
               │ 实现
      ┌────────┴────────┐
      │                 │
      ↓                 ↓
┌────────────┐  ┌────────────┐
│LiteLLM     │  │其他Provider  │
│Provider    │  │(未来扩展)    │
└────────────┘  └────────────┘
```

**好处**:
- 切换提供商不需要修改 AgentLoop
- 添加新提供商只需实现接口
- 易于单元测试（可以 mock 抽象接口）

#### 6.1.2 统一接口设计

**问题**: 不同 LLM 提供商 API 差异巨大
- Anthropic: Claude API
- OpenAI: GPT API
- Google: Gemini API

**解决方案**: 定义统一接口

```python
# 统一的请求格式
request = {
    "messages": [...],      # 所有提供商都支持
    "tools": [...],        # OpenAI Function Calling 格式
    "model": "...",       # 可能需要前缀
    "temperature": 0.7,
}

# 统一的响应格式
response = {
    "content": "...",
    "tool_calls": [...],
    "finish_reason": "stop",
    "usage": {...}
}
```

**实现技术**: 使用适配器库（LiteLLM）进行转换

#### 6.1.3 错误处理策略

```python
try:
    response = await acompletion(**kwargs)
    return self._parse_response(response)
except Exception as e:
    # 不抛出异常，返回错误响应
    return LLMResponse(
        content=f"Error calling LLM: {str(e)}",
        finish_reason="error",
    )
```

**设计考虑**:
- **优雅降级**: 返回有意义的错误信息而非崩溃
- **错误传播**: 错误通过 `content` 字段传递到用户
- **日志记录**: 使用 `logger.error()` 记录详细错误
- **类型安全**: 返回类型始终是 `LLMResponse`

### 6.2 编程技巧

#### 6.2.1 异步编程 (Async/Await)

```python
async def chat(self, ...) -> LLMResponse:
    # 使用异步 HTTP 客户端
    response = await acompletion(**kwargs)
    return self._parse_response(response)
```

**关键点**:
- 所有 I/O 操作都是异步的（不阻塞事件循环）
- 使用 `async/await` 语法
- 支持高并发（多个请求同时进行）

#### 6.2.2 类型注解 (Type Hints)

```python
from typing import Any

async def chat(
    self,
    messages: list[dict[str, Any]],
    tools: list[dict[str, Any]] | None = None,
    model: str | None = None,
) -> LLMResponse:
```

**Python 3.11+ 语法**:
- `list[dict[str, Any]]` 而非 `List[Dict[str, Any]]`
- `str | None` 而非 `Optional[str]`

#### 6.2.3 数据类最佳实践

```python
@dataclass
class LLMResponse:
    content: str | None
    tool_calls: list[ToolCallRequest] = field(default_factory=list)
    #                                     ^^^^^^^^^^^^^^^^^^^^
    #                                     重要：避免可变默认值问题
```

**为什么需要 default_factory**:
```python
# 错误方式：共享列表
class Bad:
    items = []  # 所有实例共享同一个列表！

# 正确方式：每个实例独立列表
class Good:
    items = field(default_factory=list)  # 每次创建新列表
```

#### 6.2.4 环境变量配置

```python
# 使用 setdefault 避免覆盖已有配置
os.environ.setdefault("ANTHROPIC_API_KEY", api_key)
#            ^^^^^^^^^
#            如果环境变量已存在，不修改
```

**用途**:
- 开发时可以在系统环境变量中设置 API 密钥
- 代码提供默认值作为后备
- 生产环境可以通过 CI/CD 注入密钥

#### 6.2.5 条件前缀逻辑

```python
# 检测提供商并添加前缀
if self.is_openrouter and not model.startswith("openrouter/"):
    model = f"openrouter/{model}"
elif "gemini" in model.lower() and not model.startswith("gemini/"):
    model = f"gemini/{model}"
```

**技巧**:
- `model.lower()` 进行不区分大小写匹配
- 使用 `startswith()` 避免重复添加前缀
- 链式 `if-elif` 提供多个选项

### 6.3 扩展性设计

#### 6.3.1 如何添加新的 LLM 提供商

**步骤 1**: 创建新的 Provider 类

```python
# nanobot/providers/custom_provider.py
from nanobot.providers.base import LLMProvider

class CustomLLMProvider(LLMProvider):
    def __init__(self, api_key: str | None = None, ...):
        super().__init__(api_key)
        # 自定义初始化

    async def chat(self, messages, tools, model, ...):
        # 调用自定义 API
        response = await custom_api_call(...)
        return self._parse_response(response)

    def get_default_model(self) -> str:
        return "custom/model-name"
```

**步骤 2**: 添加配置支持

```python
# nanobot/config/schema.py
class ProvidersConfig(BaseModel):
    # 现有提供商...
    custom: ProviderConfig = Field(default_factory=ProviderConfig)
```

**步骤 3**: 更新 CLI 以支持新提供商

```python
# nanobot/cli/commands.py
def get_api_key(self) -> str | None:
    return (
        # 现有优先级...
        self.providers.custom.api_key or
        None
    )
```

#### 6.3.2 如何添加新的 Transcription Provider

```python
# nanobot/providers/whisper_local.py
class WhisperLocalProvider:
    """本地 Whisper 模型（无需 API 密钥）"""

    def __init__(self, model_size: str = "base"):
        import whisper
        self.model = whisper.load_model(model_size)

    async def transcribe(self, file_path: str | Path) -> str:
        # 在线程池中运行（CPU 密集型任务）
        import asyncio
        loop = asyncio.get_event_loop()
        result = await loop.run_in_executor(
            None,
            self.model.transcribe,
            str(file_path)
        )
        return result["text"]
```

### 6.4 测试策略

#### 6.4.1 单元测试 Provider

```python
import pytest
from unittest.mock import AsyncMock, patch
from nanobot.providers.litellm_provider import LiteLLMProvider

@pytest.mark.asyncio
async def test_chat_returns_response():
    provider = LiteLLMProvider(api_key="test-key")

    # Mock litellm.acompletion
    with patch("nanobot.providers.litellm_provider.acompletion") as mock_acompletion:
        mock_response = AsyncMock()
        mock_response.choices = [Mock(
            message=Mock(
                content="Hello!",
                tool_calls=None
            ),
            finish_reason="stop"
        )]
        mock_response.usage = Mock(
            prompt_tokens=10,
            completion_tokens=5,
            total_tokens=15
        )
        mock_acompletion.return_value = mock_response

        response = await provider.chat(
            messages=[{"role": "user", "content": "Hi"}],
            model="test-model"
        )

        assert response.content == "Hello!"
        assert response.finish_reason == "stop"
        assert response.usage["total_tokens"] == 15
```

#### 6.4.2 Mock Provider 用于测试 Agent Loop

```python
class MockLLMProvider(LLMProvider):
    """测试用的 Mock Provider"""

    def __init__(self, responses: list[LLMResponse]):
        self.responses = iter(responses)
        self.call_count = 0

    async def chat(self, messages, tools, model, ...) -> LLMResponse:
        self.call_count += 1
        return next(self.responses)

    def get_default_model(self) -> str:
        return "mock-model"

# 使用
mock_provider = MockLLMProvider([
    LLMResponse(content="", tool_calls=[...]),  # 调用工具
    LLMResponse(content="Done!"),            # 最终响应
])
```

---

## 总结

### 核心架构特点

1. **分层设计**: 抽象层 → 适配层 → 具体实现
2. **依赖注入**: 通过构造函数传递依赖，降低耦合
3. **统一接口**: 多个提供商使用相同的调用方式
4. **错误优雅**: 错误不崩溃，返回有意义的响应
5. **扩展友好**: 添加新提供商只需实现接口

### 设计模式应用

- **抽象工厂**: 定义产品族接口
- **策略模式**: 算法可互换（不同提供商）
- **适配器模式**: 统一不同 API 格式
- **依赖倒置**: 高层依赖抽象，不依赖具体实现

### 最佳实践

1. 使用 `abc.ABC` 和 `@abstractmethod` 强制实现接口
2. 使用 `dataclass` 简化数据结构定义
3. 异步 I/O 操作（`async/await`）
4. 类型注解（Python 3.11+ 新语法）
5. 环境变量配置（`setdefault` 避免覆盖）
6. 错误日志记录（`logger.error`）
7. 优雅降级（返回错误响应而非异常）

### 可学习的关键技术

- **如何设计可扩展的架构**
- **如何统一多个外部 API**
- **如何实现函数调用（Function Calling）**
- **如何处理异步 I/O**
- **如何使用类型系统提高代码质量**
- **如何编写可测试的代码**

---

## 附录：完整代码流程图

```
┌────────────────────────────────────────────────────────────┐
│                   CLI 命令启动                        │
│  nanobot gateway / nanobot agent                    │
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
            ┌───────────────┐
            │  Config 加载  │
            │ ~/.nanobot/   │
            │ config.json   │
            └───────┬───────┘
                    │
                    ↓
            ┌─────────────────────────────┐
            │  创建 LiteLLMProvider     │
            │  - 设置 API 密钥        │
            │  - 设置 api_base         │
            │  - 检测提供商           │
            └───────────┬───────────────┘
                        │
                        ↓
            ┌──────────────────────────────────┐
            │     创建 AgentLoop            │
            │  - 注入 Provider            │
            │  - 注册工具                │
            │  - 加载 Context            │
            └───────────┬──────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────┐
│                  运行时流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                     │
│  1. 收到消息 (InboundMessage)                        │
│     ↓                                                │
│  2. 构建上下文 (ContextBuilder)                      │
│     - 系统提示词                                       │
│     - 历史消息                                        │
│     - 当前消息                                        │
│     - 工具定义                                        │
│     ↓                                                │
│  3. 调用 provider.chat(messages, tools)               │
│     ↓                                                │
│  4. 解析响应                                        │
│     - content: 文本响应                                 │
│     - tool_calls: 工具调用                              │
│     - finish_reason: 结束原因                            │
│     ↓                                                │
│  5. 如果有工具调用                                     │
│     - 执行工具                                         │
│     - 添加结果到消息历史                                │
│     - 回到步骤 3                                       │
│     ↓                                                │
│  6. 否则返回最终响应                                  │
│     - 发送 OutboundMessage                              │
└─────────────────────────────────────────────────────────────┘
```

---

**报告生成时间**: 2026-02-09
**分析者**: AI Assistant
**目的**: 学习架构设计与编程知识
