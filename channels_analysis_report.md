# nanobot channels 目录深度分析报告

> 分析日期: 2026年2月9日
> 项目版本: v0.1.3.post4
> 目标: 学习 nanobot\nanobot\channels 目录的架构设计和编程知识

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

`nanobot/channels` 目录负责**聊天渠道管理和消息路由**。它实现了一个可扩展的插件化架构，支持多种聊天平台（Telegram、WhatsApp、飞书/Lark），为用户提供统一的 AI 助手接入点。

### 核心功能

- **插件化架构**: 每个聊天平台是独立的通道插件
- **统一接口**: 所有通道继承自 `BaseChannel` 抽象基类
- **消息总线集成**: 通过 `MessageBus` 与 agent loop 解耦通信
- **权限控制**: 支持白名单机制，控制谁能使用机器人
- **消息格式化**: 自动转换 Markdown 为各平台支持的格式（HTML、纯文本等）
- **多媒体支持**: 支持图片、语音、文档、表情贴等多种消息类型
- **语音转录**: Telegram 集成 Groq Whisper API 自动转录语音消息
- **WebSocket 长连接**: 飞书和 WhatsApp 使用 WebSocket 无需公网 IP

---

## 目录结构

```
nanobot/
└── channels/
    ├── __init__.py        # 模块导出
    ├── base.py           # 抽象基类定义
    ├── manager.py        # 通道管理器
    ├── telegram.py       # Telegram 通道实现
    ├── whatsapp.py       # WhatsApp 通道实现
    └── feishu.py        # 飞书/Lark 通道实现
```

### 文件功能概览

| 文件 | 代码行数 | 职责 |
|------|---------|------|
| `__init__.py` | 7 | 模块导出接口 |
| `base.py` | 122 | 抽象基类和接口定义 |
| `manager.py` | 151 | 通道生命周期管理 |
| `telegram.py` | 303 | Telegram 通道实现 |
| `whatsapp.py` | 142 | WhatsApp 桥接实现 |
| `feishu.py` | 264 | 飞书 WebSocket 实现 |

**总计**: 989 行核心代码

---

## 核心架构设计

### 1. 插件化架构 (Plugin Architecture)

```
┌─────────────────────────────────────────────────┐
│          ChannelManager (管理器)          │
│  - 负责通道的创建、启动、停止        │
│  - 管理消息路由                   │
└───────────────────┬──────────────────────────┘
                    │ 管理
            ┌──────┴───────────┐
            │                      │
            ↓                      ↓
┌────────────────────────┐  ┌────────────────────────┐  ┌────────────────────────┐
│  TelegramChannel      │  │  WhatsAppChannel     │  │  FeishuChannel         │
│  - Python实现         │  │  - WebSocket桥接     │  │  - WebSocket实现        │
│  - 长轮询             │  │  - Node.js通信        │  │  - 长连接             │
│  - Markdown转HTML       │  │  - JSON通信          │  │  - Lark SDK           │
└─────────────────────────┘  └─────────────────────────┘       └─────────────────────────┘

            ↓ 统一继承自 BaseChannel
┌─────────────────────────────────────────────────┐
│              BaseChannel (抽象基类)          │
│  - 定义统一接口契约                    │
│  - 权限验证                            │
│  - 消息转发逻辑                        │
└─────────────────────────────────────────────────┘
```

**设计优势**:
- **开闭原则**: 添加新通道只需实现 `BaseChannel` 接口
- **依赖注入**: 通道通过构造函数接收 `config` 和 `bus`
- **解耦通信**: 所有通道通过 `MessageBus` 与 agent 通信
- **独立配置**: 每个通道有独立的配置类（`TelegramConfig`, `WhatsAppConfig` 等）

### 2. BaseChannel 抽象基类

```python
from abc import ABC, abstractmethod
from typing import Any

from nanobot.bus.events import InboundMessage, OutboundMessage
from nanobot.bus.queue import MessageBus


class BaseChannel(ABC):
    """
    聊天渠道的抽象基类

    每个聊天平台（Telegram, Discord, etc.)应该实现这个接口
    以集成到 nanobot 消息总线。
    """

    name: str = "base"

    def __init__(self, config: Any, bus: MessageBus):
        """
        初始化通道

        Args:
            config: 通道特定的配置
            bus: 消息总线，用于与 agent 通信
        """
        self.config = config
        self.bus = bus
        self._running = False

    @abstractmethod
    async def start(self) -> None:
        """
        启动通道并开始监听消息

        This should be a long-running async task that:
        1. Connects to the chat platform
        2. Listens for incoming messages
        3. Forwards messages to the bus via _handle_message()
        """
        pass

    @abstractmethod
    async def stop(self) -> None:
        """
        停止通道并清理资源
        """
        pass

    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None:
        """
        通过此通道发送消息

        Args:
            msg: 要发送的消息
        """
        pass

    def is_allowed(self, sender_id: str) -> bool:
        """
        检查发送者是否被允许使用机器人

        Args:
            sender_id: 发送者标识符

        Returns:
            True if allowed, False otherwise
        """
        # 获取白名单配置
        allow_list = getattr(self.config, "allow_from", [])

        # 如果没有白名单，允许所有人
        if not allow_list:
            return True

        # 转换为字符串进行匹配
        sender_str = str(sender_id)

        # 如果在白名单中
        if sender_str in allow_list:
            return True

        # 支持格式: "123456789|username"
        if "|" in sender_str:
            for part in sender_str.split("|"):
                if part and part in allow_list:
                    return True
        return False

    async def _handle_message(
        self,
        sender_id: str,
        chat_id: str,
        content: str,
        media: list[str] | None = None,
        metadata: dict[str, Any] | None = None
    ) -> None:
        """
        处理传入的消息

        此方法检查权限并将消息转发到消息总线

        Args:
            sender_id: 发送者标识
            chat_id: 聊天/频道 ID
            content: 消息文本
            media: 可选的媒体文件路径
            metadata: 可选的通道特定元数据
        """
        # 检查权限
        if not self.is_allowed(sender_id):
            return

        # 创建 InboundMessage
        msg = InboundMessage(
            channel=self.name,
            sender_id=str(sender_id),
            chat_id=str(chat_id),
            content=content,
            media=media or [],
            metadata=metadata or {}
        )

        # 转发到消息总线
        await self.bus.publish_inbound(msg)

    @property
    def is_running(self) -> bool:
        """检查通道是否正在运行"""
        return self._running
```

**关键设计点**:

1. **抽象方法**: 使用 `@abstractmethod` 强制子类实现核心方法
   - `start()` - 启动通道
   - `stop()` - 停止通道
   - `send()` - 发送消息

2. **通用方法**: `_handle_message()` 提供默认的消息处理逻辑
   - 权限检查
   - 创建 `InboundMessage`
   - 转发到消息总线

3. **权限系统**: `is_allowed()` 方法实现灵活的白名单机制
   - 支持单个 ID 匹配
   - 支持格式: `"ID|username"`

---

## 详细实现分析

### 4.1 ChannelManager - 通道管理器

```python
class ChannelManager:
    """
    管理聊天通道并协调消息路由

    职责：
    - 根据配置初始化启用的通道
    - 启动/停止所有通道
    - 将出站消息路由到正确的通道
    """

    def __init__(self, config: Config, bus: MessageBus):
        self.config = config
        self.bus = bus
        self.channels: dict[str, BaseChannel] = {}
        self._dispatch_task: asyncio.Task | None = None

    def _init_channels(self) -> None:
        """根据配置初始化通道"""
        # Telegram 通道
        if self.config.channels.telegram.enabled:
            from nanobot.channels.telegram import TelegramChannel
            self.channels["telegram"] = TelegramChannel(
                self.config.channels.telegram,
                self.bus,
                groq_api_key=self.config.providers.groq.api_key
            )

        # WhatsApp 通道
        if self.config.channels.whatsapp.enabled:
            from nanobot.channels.whatsapp import WhatsAppChannel
            self.channels["whatsapp"] = WhatsAppChannel(
                self.config.channels.whatsapp,
                self.bus
            )

        # 飞书/Lark 通道
        if self.config.channels.feishu.enabled:
            from nanobot.channels.feishu import FeishuChannel
            try:
                self.channels["feishu"] = FeishuChannel(
                    self.config.channels.feishu,
                    self.bus
                )
            except ImportError:
                logger.warning("Feishu SDK not installed")

    async def start_all(self) -> None:
        """启动所有通道"""
        if not self.channels:
            logger.warning("No channels enabled")
            return

        # 启动出站消息分发器
        self._dispatch_task = asyncio.create_task(self._dispatch_outbound())

        # 并行启动所有通道
        tasks = []
        for name, channel in self.channels.items():
            logger.info(f"Starting {name} channel...")
            tasks.append(asyncio.create_task(channel.start()))

        # 等待所有通道完成（它们应该永远运行）
        await asyncio.gather(*tasks, return_exceptions=True)

    async def stop_all(self) -> None:
        """停止所有通道"""
        logger.info("Stopping all channels...")

        # 停止分发器
        if self._dispatch_task:
            self._dispatch_task.cancel()
            try:
                await self._dispatch_task
            except asyncio.CancelledError:
                pass

        # 停止所有通道
        for name, channel in self.channels.items():
            try:
                await channel.stop()
                logger.info(f"Stopped {name} channel")
            except Exception as e:
                logger.error(f"Error stopping {name}: {e}")

    async def _dispatch_outbound(self) -> None:
        """将出站消息路由到正确的通道"""
        while True:
            try:
                msg = await asyncio.wait_for(
                    self.bus.consume_outbound(),
                    timeout=1.0
                )

                # 获取目标通道
                channel = self.channels.get(msg.channel)
                if channel:
                    await channel.send(msg)
                else:
                    logger.warning(f"Unknown channel: {msg.channel}")
            except asyncio.TimeoutError:
                continue
            except asyncio.CancelledError:
                break

    def get_channel(self, name: str) -> BaseChannel | None:
        """通过名称获取通道"""
        return self.channels.get(name)

    def get_status(self) -> dict[str, Any]:
        """获取所有通道状态"""
        return {
            name: {
                "enabled": True,
                "running": channel.is_running
            }
            for name, channel in self.channels.items()
        }

    @property
    def enabled_channels(self) -> list[str]:
        """获取已启用的通道名称列表"""
        return list(self.channels.keys())
```

**关键特性**:

1. **懒加载**: 通道仅在配置启用时才被加载和实例化
2. **错误处理**: 导入失败时记录警告，但不会阻止其他通道启动
3. **优雅关闭**: 即使某个通道停止失败，仍尝试停止其他通道
4. **循环分发**: 持续监听消息总线并路由到目标通道

---

### 4.2 TelegramChannel - Telegram 通道

```python
class TelegramChannel(BaseChannel):
    """
    Telegram 通道实现

    特点：
    - 使用 python-telegram-bot 库
    - 长轮询模式（无需 webhook）
    - 支持多种消息类型（文本、图片、语音、文档）
    - 自动 Markdown 转 HTML
    - 集成 Groq Whisper 语音转录
    """

    name = "telegram"

    def __init__(self, config, bus, groq_api_key):
        super().__init__(config, bus)
        self.config = config
        self.groq_api_key = groq_api_key
        self._app: Application | None = None
        self._chat_ids: dict[str, int] = {}  # 存储发送者ID到chat_id映射

    async def start(self):
        """启动 Telegram 机器人"""
        if not self.config.token:
            logger.error("Telegram bot token not configured")
            return

        self._running = True

        # 构建应用
        self._app = (
            Application.builder()
            .token(self.config.token)
            .build()
        )

        # 添加消息处理器（文本、图片、语音、文档）
        self._app.add_handler(
            MessageHandler(
                (filters.TEXT | filters.PHOTO | filters.VOICE | filters.AUDIO | filters.Document.ALL)
                & ~filters.COMMAND,
                self._on_message
            )
        )

        # 添加 /start 命令处理器
        from telegram.ext import CommandHandler
        self._app.add_handler(CommandHandler("start", self._on_start))

        logger.info("Starting Telegram bot (polling mode)...")

        # 初始化并启动轮询
        await self._app.initialize()
        await self._app.start()

        # 获取机器人信息
        bot_info = await self._app.bot.get_me()
        logger.info(f"Telegram bot @{bot_info.username} connected")

        # 启动轮询（阻塞运行直到停止）
        await self._app.updater.start_polling(
            allowed_updates=["message"],
            drop_pending_updates=True  # 启动时忽略旧消息
        )

        # 保持运行直到被停止
        while self._running:
            await asyncio.sleep(1)

    async def stop(self):
        """停止 Telegram 机器人"""
        self._running = False

        if self._app:
            logger.info("Stopping Telegram bot...")
            await self._app.updater.stop()
            await self._app.stop()
            await self._app.shutdown()
            self._app = None

    async def send(self, msg):
        """发送消息到 Telegram"""
        if not self._app:
            logger.warning("Telegram bot not running")
            return

        try:
            # chat_id 应该是 Telegram chat ID（整数）
            chat_id = int(msg.chat_id)

            # 转换 Markdown 为 Telegram HTML
            html_content = _markdown_to_telegram_html(msg.content)
            await self._app.bot.send_message(
                chat_id=chat_id,
                text=html_content,
                parse_mode="HTML"
            )
        except ValueError:
            logger.error(f"Invalid chat_id: {msg.chat_id}")
        except Exception as e:
            # HTML 解析失败时回退到纯文本
            logger.warning(f"HTML parse failed, falling back to plain text: {e}")
            try:
                await self._app.bot.send_message(
                    chat_id=int(msg.chat_id),
                    text=msg.content
                )
            except Exception as e2:
                logger.error(f"Error sending Telegram message: {e2}")

    async def _on_start(self, update, context):
        """处理 /start 命令"""
        if not update.message or not update.effective_user:
            return

        user = update.effective_user
        await update.message.reply_text(
            f"👋 Hi {user.first_name}! I'm nanobot.\n\nSend me a message and I'll respond!"
        )

    async def _on_message(self, update, context):
        """处理传入消息（文本、图片、语音、文档）"""
        if not update.message or not update.effective_user:
            return

        message = update.message
        user = update.effective_user
        chat_id = message.chat_id

        # 使用稳定 ID，但保留用户名用于白名单
        sender_id = str(user.id)
        if user.username:
            sender_id = f"{sender_id}|{user.username}"

        # 存储映射用于回复
        self._chat_ids[sender_id] = chat_id

        # 构建内容
        content_parts = []
        media_paths = []

        # 文本内容
        if message.text:
            content_parts.append(message.text)
        if message.caption:
            content_parts.append(message.caption)

        # 处理媒体文件
        media_file = None
        media_type = None

        if message.photo:
            media_file = message.photo[-1]  # 最大的图片
            media_type = "image"
        elif message.voice:
            media_file = message.voice
            media_type = "voice"
        elif message.audio:
            media_file = message.audio
            media_type = "audio"
        elif message.document:
            media_file = message.document
            media_type = "file"

        # 下载媒体
        if media_file and self._app:
            try:
                file = await self._app.bot.get_file(media_file.file_id)
                ext = self._get_extension(media_type, getattr(media_file, 'mime_type', None))

                # 保存到 ~/.nanobot/media/
                from pathlib import Path
                media_dir = Path.home() / ".nanobot" / "media"
                media_dir.mkdir(parents=True, exist_ok=True)
                file_path = media_dir / f"{media_file.file_id[:16]}{ext}"
                await file.download_to_drive(str(file_path))
                media_paths.append(str(file_path))

                # 语音转录
                if media_type == "voice" or media_type == "audio":
                    from nanobot.providers.transcription import GroqTranscriptionProvider
                    transcriber = GroqTranscriptionProvider(api_key=self.groq_api_key)
                    transcription = await transcriber.transcribe(file_path)
                    if transcription:
                        logger.info(f"Transcribed {media_type}: {transcription[:50]}...")
                        content_parts.append(f"[transcription: {transcription}]")
                    else:
                        content_parts.append(f"[{media_type}: {file_path}]")
            except Exception as e:
                logger.error(f"Failed to download media: {e}")
                content_parts.append(f"[{media_type}: download failed]")

        content = "\n".join(content_parts) if content_parts else "[empty message]"

        # 转发到消息总线
        await self._handle_message(
            sender_id=sender_id,
            chat_id=str(chat_id),
            content=content,
            media=media_paths,
            metadata={
                "message_id": message.message_id,
                "user_id": user.id,
                "username": user.username,
                "first_name": user.first_name,
                "is_group": message.chat.type != "private"
            }
        )

    def _get_extension(self, media_type, mime_type):
        """根据媒体类型和 MIME 类型获取文件扩展"""
        if mime_type:
            ext_map = {
                "image/jpeg": ".jpg", "image/png": ".png", "image/gif": ".gif",
                "audio/ogg": ".ogg", "audio/mpeg": ".mp3", "audio/mp4": ".m4a",
            }
            return ext_map.get(mime_type, "")
        type_map = {"image": ".jpg", "voice": ".ogg", "audio": ".mp3", "file": ""}
        return type_map.get(media_type, "")
```

**Markdown 转 HTML 转换器**（核心技巧）：

```python
def _markdown_to_telegram_html(text: str) -> str:
    """转换 Markdown 为 Telegram-safe HTML"""

    if not text:
        return ""

    # 1. 提取并保护代码块（保留内容不被其他处理修改）
    code_blocks = []
    def save_code_block(m):
        code_blocks.append(m.group(1))
        return f"\x00CB{len(code_blocks)-1}\x00"

    text = re.sub(r'```[\w]*\n?([\s\S]*?)```', save_code_block, text)

    # 2. 提取并保护内联代码
    inline_codes = []
    def save_inline_code(m):
        inline_codes.append(m.group(1))
        return f"\x00IC{len(inline_codes)-1}\x00"

    text = re.sub(r'`([^`]+)`', save_inline_code, text)

    # 3. 删除标题标记
    text = re.sub(r'^#{1,6}\s+(.+)$', r'\1', text, flags=re.MULTILINE)

    # 4. 删除引用块标记
    text = re.sub(r'^>\s*(.*)$', r'\1', text, flags=re.MULTILINE)

    # 5. 转义 HTML 特殊字符
    text = text.replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;")

    # 6. 处理链接 [text](url)
    text = re.sub(r'\[([^\]]+)\]\(([^)]+)\)', r'<a href="\2">\1</a>', text)

    # 7. 处理粗体 **text** 或 __text__
    text = re.sub(r'\*\*(.+?)\*\*', r'<b>\1</b>', text)
    text = re.sub(r'__(.+?)__', r'<b>\1</b>', text)

    # 8. 处理斜体 _text_（避免匹配 inside words）
    text = re.sub(r'(?<![a-zA-Z0-9])_([^_]+)_(?![a-zA-Z0-9])', r'<i>\1</i>', text)

    # 9. 处理删除线 ~~text~~
    text = re.sub(r'~~(.+?)~~', r'<s>\1</s>', text)

    # 10. 处理无序列列表 - item -> • item
    text = re.sub(r'^[-*]\s+', '• ', text, flags=re.MULTILINE)

    # 11. 恢复内联代码（带 HTML 标签）
    for i, code in enumerate(inline_codes):
        escaped = code.replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;")
        text = text.replace(f"\x00IC{i}\x00", f"<code>{escaped}</code>")

    # 12. 恢复代码块（带 pre/code 标签）
    for i, code in enumerate(code_blocks):
        escaped = code.replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;")
        text = text.replace(f"\x00CB{i}\x00", f"<pre><code>{escaped}</code></pre>")

    return text
```

**正则表达式技巧**：
- `r'```[\w]*\n?([\s\S]*?)```'` - 使用非贪婪匹配 `[\w]*` 允许代码块语言标识
- `flags=re.MULTILINE` - 使 `^` 和 `$` 能匹配每行开头/结尾
- `(?<![a-zA-Z0-9])` - 使用负向前查找避免匹配单词内部的下划线
- `f"\x00CB{len(code_blocks)-1}\x00"` - 使用自定义格式标记代码块位置

---

### 4.3 WhatsAppChannel - WhatsApp 桥接

```python
class WhatsAppChannel(BaseChannel):
    """
    WhatsApp 通道实现

    特点：
    - 通过 WebSocket 连接到 Node.js 桥接服务
    - 桥接使用 @whiskeysockets/baileys 库处理 WhatsApp 协议
    - 支持 JSON 格式的消息通信
    - 语音消息目前不支持直接转录
    """

    name = "whatsapp"

    def __init__(self, config, bus):
        super().__init__(config, bus)
        self.config = config
        self._ws = None
        self._connected = False

    async def start(self):
        """启动 WhatsApp 通道"""
        import websockets

        bridge_url = self.config.bridge_url

        logger.info(f"Connecting to WhatsApp bridge at {bridge_url}...")
        self._running = True

        # WebSocket 连接循环（带重连）
        while self._running:
            try:
                async with websockets.connect(bridge_url) as ws:
                    self._ws = ws
                    self._connected = True
                    logger.info("Connected to WhatsApp bridge")

                    # 监听消息
                    async for message in ws:
                        await self._handle_bridge_message(message)

            except asyncio.CancelledError:
                break
            except Exception as e:
                self._connected = False
                self._ws = None
                logger.warning(f"WhatsApp bridge connection error: {e}")

                if self._running:
                    logger.info("Reconnecting in 5 seconds...")
                    await asyncio.sleep(5)

    async def stop(self):
        """停止 WhatsApp 通道"""
        self._running = False
        self._connected = False

        if self._ws:
            await self._ws.close()
            self._ws = None

    async def send(self, msg):
        """发送消息到 WhatsApp"""
        if not self._ws or not self._connected:
            logger.warning("WhatsApp bridge not connected")
            return

        try:
            payload = {
                "type": "send",
                "to": msg.chat_id,
                "text": msg.content
            }
            await self._ws.send(json.dumps(payload))
        except Exception as e:
            logger.error(f"Error sending WhatsApp message: {e}")

    async def _handle_bridge_message(self, raw: str):
        """处理来自桥接的消息"""
        try:
            data = json.loads(raw)
        except json.JSONDecodeError:
            logger.warning(f"Invalid JSON from bridge: {raw[:100]}")
            return

        msg_type = data.get("type")

        if msg_type == "message":
            # 传入消息
            sender = data.get("sender", "")
            content = data.get("content", "")

            # 发送者格式: <phone>@s.whatsapp.net
            # 提取手机号作为 chat_id
            chat_id = sender.split("@")[0] if "@" in sender else sender

            await self._handle_message(
                sender_id=chat_id,
                chat_id=sender,  # 使用完整 JID 用于回复
                content=content,
                metadata={
                    "message_id": data.get("id"),
                    "timestamp": data.get("timestamp"),
                    "isGroup": data.get("isGroup", False)
                }
            )

        elif msg_type == "status":
            # 连接状态更新
            status = data.get("status")
            logger.info(f"WhatsApp status: {status}")

            if status == "connected":
                self._connected = True
            elif status == "disconnected":
                self._connected = False

        elif msg_type == "qr":
            # QR 码用于认证
            logger.info("Scan QR code in bridge terminal to connect WhatsApp")
```

**Node.js 桥接协议示例**：

```json
// 桥接发送的 JSON 消息格式
{
  "type": "message",     // 消息类型
  "sender": "123456789@c.us",  // 发送者
  "id": "3EB0...",              // 消息 ID
  "content": "Hello!",          // 消息内容
  "timestamp": "1234567890",  // 时间戳
  "isGroup": false           // 是否群组
}

// 桥接接收的发送命令格式
{
  "type": "send",
  "to": "123456789@c.us",  // 接收者
  "text": "Hi there!"          // 要发送的内容
}
```

**通信流程**：
```
Telegram/WhatsApp/Lark 用户
    ↓
python-telegram-bot / Node.js 桥接 / Lark WebSocket
    ↓
WebSocket 长连接
    ↓
Python nanobot channels (通过 WebSocket 或 HTTP)
    ↓
MessageBus (消息队列)
    ↓
AgentLoop (AI 处理)
    ↓
MessageBus (发送响应)
    ↓
Telegram/WhatsApp/Lark (回复用户)
```

---

### 4.4 FeishuChannel - 飞书/Lark 通道

```python
class FeishuChannel(BaseChannel):
    """
    飞书/Lark 通道实现

    特点：
    - 使用 lark-oapi SDK
    - WebSocket 长连接（无需公网 IP）
    - 支持消息、图片、文件、表情包
    - 支持添加反应（表情回应）
    - 消息去重机制（最多缓存 1000 条已处理消息）
    - 支持群组和私聊区分
    """

    name = "feishu"

    def __init__(self, config, bus):
        super().__init__(config, bus)
        self.config = config
        self._client: Any = None
        self._ws_client: Any = None
        self._processed_message_ids: OrderedDict[str, None] = OrderedDict()  # 去重缓存
        self._loop: asyncio.AbstractEventLoop | None = None

    async def start(self):
        """启动飞书机器人"""
        try:
            from lark_oapi import lark
        except ImportError:
            logger.error("Feishu SDK not installed. Run: pip install lark-oapi")
            return

        if not self.config.app_id or not self.config.app_secret:
            logger.error("Feishu app_id and app_secret not configured")
            return

        self._running = True
        self._loop = asyncio.get_running_loop()

        # 创建 Lark 客户端
        self._client = lark.Client.builder() \
            .app_id(self.config.app_id) \
            .app_secret(self.config.app_secret) \
            .log_level(lark.LogLevel.INFO) \
            .build()

        # 创建事件处理器（只注册消息接收）
        event_handler = lark.EventDispatcherHandler.builder(
            self.config.encrypt_key or "",
            self.config.verification_token or "",
        ).register_p2_im_message_receive_v1(
            self._on_message_sync
        ).build()

        # 创建 WebSocket 客户端（用于接收事件）
        self._ws_client = lark.ws.Client(
            self.config.app_id,
            self.config.app_secret,
            event_handler=event_handler,
            log_level=lark.LogLevel.INFO
        )

        # 在单独线程中运行 WebSocket（避免阻塞事件循环）
        def run_ws():
            try:
                self._ws_client.start()
            except Exception as e:
                logger.error(f"Feishu WebSocket error: {e}")

        from threading import Thread
        self._ws_thread = Thread(target=run_ws, daemon=True)
        self._ws_thread.start()

        logger.info("Feishu bot started with WebSocket long connection")

    async def stop(self):
        """停止飞书机器人"""
        self._running = False

        if self._ws_client:
            try:
                await self._ws_client.stop()
            except Exception as e:
                logger.warning(f"Error stopping WebSocket client: {e}")

        logger.info("Feishu bot stopped")

    async def send(self, msg):
        """发送消息到飞书"""
        if not self._client:
            logger.warning("Feishu client not initialized")
            return

        try:
            # 确定 receive_id_type（open_id 或 chat_id）
            if msg.chat_id.startswith("oc_"):
                receive_id_type = "chat_id"
            else:
                receive_id_type = "open_id"

            # 构建文本消息内容
            content = json.dumps({"text": msg.content})

            # 创建消息请求
            request = CreateMessageRequest.builder() \
                .receive_id_type(receive_id_type) \
                .receive_id(msg.chat_id) \
                .msg_type("text") \
                .request_body(
                    CreateMessageRequestBody.builder()
                    .receive_id(msg.chat_id)
                    .msg_type("text")
                    .content(content)
                    .build()
                ).build()

            # 发送消息
            response = self._client.im.v1.message.create(request)

            if not response.success():
                logger.error(
                    f"Failed to send Feishu message: code={response.code}, "
                    f"msg={response.msg}, log_id={response.get_log_id()}"
                )
            else:
                logger.debug(f"Feishu message sent to {msg.chat_id}")

        except Exception as e:
            logger.error(f"Error sending Feishu message: {e}")

    def _add_reaction_sync(self, message_id: str, emoji_type: str):
        """同步添加反应表情（在线程池中执行）"""
        try:
            request = CreateMessageReactionRequest.builder() \
                .message_id(message_id) \
                .request_body(
                    CreateMessageReactionRequestBody.builder()
                    .reaction_type(Emoji.builder().emoji_type(emoji_type)).build()
                ).build()

            response = self._client.im.v1.message_reaction.create(request)

            if not response.success():
                logger.warning(f"Failed to add reaction: code={response.code}")
            else:
                logger.debug(f"Added {emoji_type} reaction to message {message_id}")
        except Exception as e:
            logger.warning(f"Error adding reaction: {e}")

    async def _add_reaction(self, message_id: str, emoji_type: str):
        """异步添加反应"""
        if not self._client or not Emoji:
            return

        loop = asyncio.get_running_loop()
        await loop.run_in_executor(None, self._add_reaction_sync, message_id, emoji_type)

    async def _on_message_sync(self, data):
        """
        同步处理传入消息（由 WebSocket 线程调用）

        此方法安排异步处理到主事件循环
        """
        if self._loop and self._loop.is_running():
            asyncio.run_coroutine_threadsafe(self._on_message, data)

    async def _on_message(self, data):
        """处理飞书消息"""
        try:
            event = data.event
            message = event.message
            sender = event.sender
            message_id = message.message_id

            # 去重检查
            if message_id in self._processed_message_ids:
                return

            self._processed_message_ids[message_id] = None

            # 修剪缓存（最多 1000 条）
            while len(self._processed_message_ids) > 1000:
                self._processed_message_ids.popitem(last=False)

            # 跳过机器人消息
            sender_type = sender.sender_type
            if sender_type == "bot":
                return

            # 获取发送者信息
            sender_id = sender.sender.open_id if sender.sender_type == "open_id" else "unknown"
            chat_id = message.chat_id
            chat_type = message.chat_type  # "p2p" 或 "group"

            # 解析消息内容
            if chat_type == "text":
                try:
                    content = json.loads(message.content).get("text", "")
                except json.JSONDecodeError:
                    content = message.content or ""

            elif chat_type in ["image", "audio", "file", "sticker"]:
                content = MSG_TYPE_MAP.get(chat_type, f"[{chat_type}]")

            if not content:
                return

            # 添加"已读"反应表示消息已处理
            await self._add_reaction(message_id, "EYES")

            # 转发到消息总线
            reply_to = chat_id if chat_type == "group" else sender_id
            await self._handle_message(
                sender_id=sender_id,
                chat_id=reply_to,
                content=content,
                metadata={
                    "message_id": message_id,
                    "chat_type": chat_type,
                    "msg_type": chat_type
                }
            )

    # 消息类型显示映射
    MSG_TYPE_MAP = {
        "image": "[image]",
        "audio": "[audio]",
        "file": "[file]",
        "sticker": "[sticker]"
    }
```

**WebSocket 长连接优势**：
- 无需公网 IP：适合无服务器环境（本地开发）
- 实时双向通信：比 webhook 更快响应
- 稳定连接：自动重连机制
- 事件驱动：支持服务器推送事件（新消息、状态变化等）

---

## 集成模式

### 5.1 消息流转图

```
┌────────────────────────────────────────────────────────────┐
│              用户发送消息                              │
│  (Telegram/WhatsApp/Feishu)                   │
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│         Channel 创建 InboundMessage                  │
│  - channel: "telegram"                           │
│  - sender_id: 用户 ID                             │
│  - chat_id: 聊天 ID                            │
│  - content: 消息内容                           │
│  - media: 媒体文件路径（可选）                 │
│  - metadata: 通道特定信息                          │
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│         Channel._handle_message()                     │
│  - 检查权限 (is_allowed)                     │
│  - 权限允许 → 创建 InboundMessage                 │
│  - 权限拒绝 → 返回，不转发                   │
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│         MessageBus.publish_inbound()                  │
│  - 将消息推入 inbound 队列                   │
│  - 队列: asyncio.Queue[InboundMessage]        │
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│         AgentLoop.run()                             │
│  - await bus.consume_inbound()                     │
│  - 获取消息，处理                           │
│  - 调用 provider.chat()                          │
│  - 执行工具                                   │
│  - 创建 OutboundMessage                          │
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│         MessageBus.publish_outbound()                   │
│  - 将响应推入 outbound 队列                 │
│  - 队列: asyncio.Queue[OutboundMessage]        │
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│         ChannelManager._dispatch_outbound()             │
│  - await bus.consume_outbound()                    │
│  - 获取目标 channel                           │
│  - channel.send(msg)                            │
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│         用户收到响应                                 │
└─────────────────────────────────────────────────────────┘
```

---

### 5.2 ChannelManager 与 CLI 集成

```python
# cli/commands.py 中的使用

def gateway():
    # 创建消息总线
    bus = MessageBus()

    # 创建会话管理器
    sessions = SessionManager(workspace)

    # 创建 provider
    provider = LiteLLMProvider(
        api_key=config.get_api_key(),
        api_base=config.get_api_base(),
        default_model=config.agents.defaults.model
    )

    # 创建 agent
    agent = AgentLoop(
        bus=bus,
        provider=provider,
        workspace=config.workspace_path,
        model=config.agents.defaults.model,
        max_iterations=config.agents.defaults.max_tool_iterations,
        brave_api_key=config.tools.web.search.api_key or None,
        exec_config=config.tools.exec,
    )

    # 创建通道管理器
    channels = ChannelManager(config, bus)

    # 启动所有服务
    async def run():
        await cron.start()
        await heartbeat.start()
        await asyncio.gather(
            agent.run(),
            channels.start_all(),
        )
```

---

## 设计模式总结

### 6.1 使用的设计模式

| 模式 | 位置 | 用途 |
|------|------|------|
| **抽象工厂 (Abstract Factory)** | `BaseChannel` | 定义产品族接口 |
| **策略模式 (Strategy)** | 各通道实现 | 不同通信策略（长轮询、WebSocket、桥接） |
| **适配器模式 (Adapter)** | `MessageBus` | 解耦通信机制 |
| **观察者模式 (Observer)** | `ChannelManager` | 管理通道生命周期 |
| **单例模式 (Singleton)** | `MessageBus` | 共享消息队列 |
| **依赖注入 (Dependency Injection)** | 构造函数注入 `config` 和 `bus` |
| **模板方法 (Template Method)** | `_handle_message()` | 定义消息处理流程 |
| **工厂方法 (Factory Method)** | `_init_channels()` | 按需创建通道实例 |

### 6.2 架构原则体现

1. **开闭原则 (Open/Closed)**:
   - 对扩展开放：添加新通道只需实现 `BaseChannel` 接口
   - 对修改关闭：现有通道管理器无需修改

2. **里氏替换原则 (Liskov Substitution)**:
   - 任何 `BaseChannel` 实现都可以互换使用
   - `AgentLoop` 和 `ChannelManager` 不关心具体是哪个通道

3. **单一职责原则 (Single Responsibility)**:
   - `BaseChannel`: 只定义接口
   - `TelegramChannel`: 处理 Telegram 特定逻辑
   - `WhatsAppChannel`: 处理 WhatsApp 通信
   - `ChannelManager`: 管理通道生命周期

4. **接口隔离原则 (Interface Segregation)**:
   - 每个通道只暴露必要的方法
   - 消息总线解耦通道与 agent

5. **依赖倒置原则 (Dependency Inversion)**:
   - 高层模块依赖抽象接口（`BaseChannel`, `MessageBus`）
   - 不依赖具体实现

---

## 学习要点

### 7.1 架构设计知识

#### 7.1.1 插件化架构的优势

**对比：单体架构 vs 插件化架构**

| 维度 | 单体架构 | 插件化架构 |
|------|---------|-------------|
| **代码耦合** | 高耦合 - 修改一个平台影响其他 | 低耦合 - 各平台独立 |
| **可测试性** | 难以单独测试单个平台 | 易于单元测试每个通道 |
| **可扩展性** | 添加新平台需要修改核心代码 | 实现接口即可添加 |
| **部署灵活性** | 必须部署全部代码 | 可选择性部署通道 |
| **团队协作** | 多人同时修改核心容易冲突 | 不同人员负责不同平台 |

#### 7.1.2 异步编程模式

**Python 异步编程核心概念**：

```python
# 异步上下文管理
import asyncio

# 创建事件循环
loop = asyncio.get_event_loop()

# 运行协程任务
async def main():
    # 并行启动多个长时间运行任务
    await asyncio.gather(
        channel.start(),
        agent.run(),
        other_task(),
    )

# 取消任务
task = asyncio.create_task(some_async_function())
await task.cancel()  # 优雅取消
```

**异步任务控制**:
- `asyncio.create_task()` - 创建后台任务
- `asyncio.gather()` - 等待多个任务完成
- `asyncio.wait_for()` - 等待单个任务
- `asyncio.sleep()` - 暂停执行

#### 7.1.3 消息队列设计

```python
# MessageBus 实现
class MessageBus:
    def __init__(self):
        self.inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()
        self.outbound: asyncio.Queue[OutboundMessage] = asyncio.Queue()

    # 生产者模式
async def publish_inbound(self, msg: InboundMessage):
    await self.inbound.put(msg)

# 消费者模式
async def consume_inbound(self) -> InboundMessage:
    return await self.inbound.get()

# 分发器模式
async def dispatch_outbound(self):
    while True:
        msg = await self.outbound.get()  # 阻塞获取
        channel = self.channels.get(msg.channel)
        if channel:
            await channel.send(msg)  # 路由到正确通道
```

**队列模式选择**:
- `asyncio.Queue()` - 内存队列，适合异步场景
- 优势**：
  - 线程安全
  - 支持背压（自动暂停生产者）
  - 异步友好的 API

#### 7.1.4 正则表达式高级技巧

**Telegram Markdown 转 HTML 使用的关键正则**：

| 正则 | 用途 | 解释 |
|------|------|------|
| ``r'```[\w]*\n?([\s\S]*?)```'` | 匹配代码块 | 非贪婪匹配语言标识 |
| `r'`([^`]+)`'` | 匹配内联代码 | 匹配反引号内的内容 |
| `flags=re.MULTILINE` | 使 `^` 和 `$` 能匹配每行 | 处理多行文本 |
| `r'(?<![a-zA-Z0-9])'` | 负向向前查找 | 避免匹配单词内部的下划线 |
| `r'\x00CB{len(code_blocks)-1}\x00'` | 自定义标记 | 标记代码块位置用于后续恢复 |

#### 7.1.5 WebSocket 编程模式

```python
import asyncio
import websockets

async def websocket_client():
    # 建立 WebSocket 连接
    async with websockets.connect(url) as ws:
        # 监听消息
        async for message in ws:
            data = json.loads(await ws.recv())
            await handle_message(data)

    # 长连接循环
    while True:
        try:
            # 接收消息
            await handle_message()
        except websockets.exceptions.ConnectionClosed:
            logger.info("Connection closed, reconnecting...")
            await asyncio.sleep(5)  # 等待后重连
            # 重新连接
            async with websockets.connect(url) as ws:
                logger.info("Reconnected")
                continue
```

**WebSocket 连接管理**：
- **心跳机制**: 发送 ping/pong 保持连接活跃
- **重连策略**: 连接断开后自动重连
- **超时处理**: 设置合理的超时时间
- **错误处理**: 捕获连接异常并优雅处理

---

## 扩展指南

### 8.1 添加新的聊天平台

要添加新的聊天平台支持，只需实现 `BaseChannel` 接口：

```python
# 示例：添加 Discord 通道
import discord
from discord.ext import commands
from nanobot.channels.base import BaseChannel
from nanobot.bus.events import InboundMessage, OutboundMessage

class DiscordChannel(BaseChannel):
    """Discord 通道实现"""

    name = "discord"

    def __init__(self, config: Any, bus: MessageBus):
        super().__init__(config, bus)
        self.config = config
        self._client = None

    async def start(self):
        """启动 Discord bot"""
        # Discord 使用令牌认证
        intents = discord.Intents.default()
        intents.messages = True
        intents.message_content = True

        self._client = discord.Client(intents=intents)

        @self._client.event
        async def on_ready():
            print(f"Discord bot logged in as {self._client.user}")

        @self._client.event
        async def on_message(message):
            # 创建 InboundMessage
            inbound = InboundMessage(
                channel=self.name,
                sender_id=str(message.author.id),
                chat_id=str(message.channel.id),
                content=message.content
            )
            await self.bus.publish_inbound(inbound)

        await self._client.start(self.config.token)

    async def stop(self):
        """停止 Discord bot"""
        await self._client.close()

    async def send(self, msg: OutboundMessage):
        """发送消息到 Discord"""
        if self._client:
            # 获取频道对象
            channel = self._client.get_channel(int(msg.chat_id))
            await channel.send(msg.content)
```

### 8.2 添加消息去重机制

当前实现仅在 Feishu 中有，可以将其提取为通用特性：

```python
class BaseChannel:
    def __init__(self, config, bus):
        self.config = config
        self.bus = bus
        self._running = False
        # 添加去重缓存
        self._processed_ids: set = set()

    async def _handle_message(
        self,
        sender_id: str,
        chat_id: str,
        content: str,
        media: list[str] | None = None,
        metadata: dict[str, Any] | None = None
    ) -> None:
        # 去重检查
        message_id = metadata.get("message_id", "")
        if message_id in self._processed_ids:
            logger.debug(f"Duplicate message ignored: {message_id}")
            return

        # 标记为已处理
        self._processed_ids.add(message_id)

        # 继续原有逻辑
        msg = InboundMessage(...)
        await self.bus.publish_inbound(msg)
```

### 8.3 添加消息持久化

当前实现中，响应消息不会自动保存到会话历史。可以在 `BaseChannel` 中添加：

```python
async def send(self, msg: OutboundMessage):
    """发送消息并保存到会话"""
    await self._handle_outbound_to_bus(msg)
    await self._save_to_history(msg)

async def _save_to_history(self, msg: OutboundMessage):
    """保存出站消息到历史记录"""
    from nanobot.session import SessionManager

    sessions = getattr(self.bus, "sessions", None)
    if sessions:
        # 获取会话
        session = sessions.get_or_create(f"{msg.channel}:{msg.chat_id}")

        # 添加助手消息
        session.add_message("assistant", msg.content)
        sessions.save(session)
```

---

## 总结

### 核心架构特点

1. **插件化架构**: 每个聊天平台独立实现，通过 `BaseChannel` 接口统一
2. **消息总线模式**: 使用 `asyncio.Queue` 解耦通道与 agent 通信
3. **权限控制**: 统一的白名单机制，支持 ID 和 "ID|username" 格式
4. **多种通信方式**: 长轮询、WebSocket、Node.js 桥接
5. **消息格式化**: Telegram 使用 Markdown 转 HTML，其他使用纯文本或 JSON
6. **多媒体支持**: 图片、语音、文档、表情包全面支持
7. **语音转录**: Telegram 集成 Groq Whisper API
8. **去重机制**: Feishu 支持消息去重，避免重复处理
9. **错误处理**: 每个组件都有完善的异常处理和日志记录
10. **优雅关闭**: 支持热停止所有通道

### 设计模式应用

- **抽象工厂**: `BaseChannel` 定义产品族接口
- **策略模式**: 不同通道使用不同通信策略
- **适配器模式**: 消息总线适配不同通道
- **观察者模式**: 通道管理器观察并管理所有通道
- **模板方法**: `_handle_message` 提供默认处理流程
- **依赖注入**: 构造函数注入依赖，便于测试

### 最佳实践

1. **抽象基类设计**: 使用 `@abstractmethod` 强制子类实现必要方法
2. **消息队列**: 使用 `asyncio.Queue` 实现生产者-消费者模式
3. **异步编程**: 正确使用 `async/await` 而非阻塞操作
4. **正则表达式**: Telegram 的 Markdown 转换展示了高级正则技巧
5. **WebSocket 长连接**: 实现自动重连和错误恢复
6. **权限控制**: 白名单机制确保安全性
7. **错误处理**: 记录详细日志，提供优雅降级

### 可学习的关键技术

- **如何设计插件化架构**
- **如何实现抽象基类和使用多态**
- **消息总线模式的生产者-消费者设计**
- **异步编程的最佳实践**（协程、任务管理、并发控制）
- **正则表达式的高级用法**（非贪婪匹配、多行处理、自定义标记）
- **WebSocket 长连接的实现技巧**
- **跨语言通信**（Python ↔ Node.js）
- **消息去重和缓存策略**

---

**报告生成时间**: 2026-02-09
**分析者**: AI Assistant
**目的**: 学习架构设计与编程知识
