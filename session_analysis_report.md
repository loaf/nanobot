# nanobot session 目录深度分析报告

> 分析日期: 2026年2月9日
> 项目版本: v0.1.3.post4
> 目标: 学习 nanobot\session 目录的架构设计和编程知识

---

## 目录

1. [概述](#概述)
2. [目录结构](#目录结构)
3. [核心架构设计](#核心架构设计)
4. [详细实现分析](#详细实现分析)
5. [数据流转](#数据流转)
6. [设计模式总结](#设计模式总结)
7. [学习要点](#学习要点)
8. [扩展指南](#扩展指南)

---

## 概述

`nanobot/session` 目录负责**会话管理和对话历史持久化**。它提供了一个轻量级但功能完整的会话管理系统，支持跨不同聊天渠道（Telegram、WhatsApp、Feishu、CLI）的对话上下文保持。

### 核心功能

- **会话创建与管理**: 为每个用户/渠道组合创建独立会话
- **对话历史存储**: 保存用户与助手的完整对话记录
- **持久化存储**: 使用 JSONL 格式（JSON Lines）存储到磁盘
- **内存缓存**: 提供快速的内存访问层
- **上下文构建**: 为 LLM 提供格式化的历史消息
- **会话列表**: 支持查询所有现有会话

---

## 目录结构

```
nanobot/
└── session/
    ├── __init__.py       # 模块导出
    └── manager.py       # 会话管理器实现
```

### 文件功能概览

| 文件 | 代码行数 | 职责 |
|------|---------|------|
| `__init__.py` | 6 | 模块导出接口 |
| `manager.py` | 203 | 会话管理核心实现 |

**总计**: 209 行核心代码

---

## 核心架构设计

### 1. 双层架构设计

```
┌─────────────────────────────────────────────────────────┐
│              SessionManager (管理器)              │
│  - 负责会话的创建、加载、保存            │
│  - 管理内存缓存                             │
│  - 处理文件系统操作                         │
└──────────────────────┬──────────────────────────┘
                       │ 管理
           ┌───────────┴───────────┐
           │                       │
           ↓                       ↓
┌─────────────────┐  ┌─────────────────┐
│  Session A      │  │  Session B      │
│  (数据结构)     │  │  (数据结构)     │
│  - key         │  │  - key         │
│  - messages    │  │  - messages    │
│  - timestamps  │  │  - timestamps  │
└─────────────────┘  └─────────────────┘
```

**设计优势**:
- **关注点分离**: 管理器处理 I/O，Session 处理数据逻辑
- **内存缓存**: 避免重复加载同一会话
- **简单存储**: JSONL 格式易于读写和调试

### 2. Session 数据类

```python
@dataclass
class Session:
    """
    会话数据结构

    以 JSONL 格式存储消息，便于读取和持久化
    """

    key: str                                  # 会话唯一标识符 (channel:chat_id)
    messages: list[dict[str, Any]] = field(default_factory=list)  # 消息列表
    created_at: datetime = field(default_factory=datetime.now)   # 创建时间
    updated_at: datetime = field(default_factory=datetime.now)   # 最后更新时间
    metadata: dict[str, Any] = field(default_factory=dict)  # 额外元数据
```

**字段说明**:

| 字段 | 类型 | 用途 |
|------|------|------|
| `key` | str | 会话唯一标识，格式为 `"channel:chat_id"` |
| `messages` | list | 对话消息列表，每个消息包含 role, content, timestamp |
| `created_at` | datetime | 会话首次创建时间 |
| `updated_at` | datetime | 最后一次添加消息的时间 |
| `metadata` | dict | 额外的会话元数据（可扩展） |

### 3. SessionManager 管理器

```python
class SessionManager:
    """
    会话管理器

    会话以 JSONL 文件形式存储在 sessions 目录
    """

    def __init__(self, workspace: Path):
        self.workspace = workspace
        self.sessions_dir = ensure_dir(Path.home() / ".nanobot" / "sessions")
        self._cache: dict[str, Session] = {}  # 内存缓存
```

**关键组件**:

1. **会话目录**: `~/.nanobot/sessions/`
2. **内存缓存**: `self._cache` 字典，避免重复读取文件
3. **工具函数**: 使用 `ensure_dir()` 和 `safe_filename()` 辅助函数

---

## 详细实现分析

### 4.1 Session 类方法

#### 4.1.1 add_message - 添加消息

```python
def add_message(self, role: str, content: str, **kwargs: Any) -> None:
    """添加消息到会话"""
    msg = {
        "role": role,
        "content": content,
        "timestamp": datetime.now().isoformat(),
        **kwargs  # 额外字段（如 tool_calls）
    }
    self.messages.append(msg)
    self.updated_at = datetime.now()
```

**设计要点**:
- **自动时间戳**: 每条消息自动添加 ISO 格式时间戳
- **灵活参数**: `**kwargs` 允许扩展消息字段
- **更新时间戳**: 每次添加消息自动更新 `updated_at`

**使用示例**:
```python
session.add_message("user", "Hello, how are you?")
session.add_message("assistant", "I'm doing well, thanks for asking!")
```

#### 4.1.2 get_history - 获取历史

```python
def get_history(self, max_messages: int = 50) -> list[dict[str, Any]]:
    """
    获取用于 LLM 上下文的消息历史

    Args:
        max_messages: 返回的最大消息数

    Returns:
        LLM 格式的消息列表
    """
    # 获取最近的消息
    recent = self.messages[-max_messages:] if len(self.messages) > max_messages else self.messages

    # 转换为 LLM 格式（只包含 role 和 content）
    return [{"role": m["role"], "content": m["content"]} for m in recent]
```

**关键设计**:

1. **滑动窗口**: 使用切片 `[-max_messages:]` 获取最近 N 条消息
   - 避免超出 LLM 上下文窗口限制
   - 默认限制为 50 条消息

2. **格式转换**: 移除内部字段（timestamp, 等），只保留 LLM 需要的 `role` 和 `content`
   - 减少不必要的 token 消耗
   - 隐藏实现细节

**窗口滑动示例**:
```
完整消息: [m1, m2, m3, m4, m5, m6, m7, m8]
max_messages=5
返回:   [m4, m5, m6, m7, m8]  # 最近5条
```

#### 4.1.3 clear - 清空会话

```python
def clear(self) -> None:
    """清空会话中的所有消息"""
    self.messages = []
    self.updated_at = datetime.now()
```

**用途**: 提供会话重置功能（虽然当前代码中未使用）

---

### 4.2 SessionManager 方法

#### 4.2.1 _get_session_path - 获取文件路径

```python
def _get_session_path(self, key: str) -> Path:
    """获取会话的文件路径"""
    safe_key = safe_filename(key.replace(":", "_"))
    return self.sessions_dir / f"{safe_key}.jsonl"
```

**文件名处理**:

1. **冒号替换**: `"channel:chat_id"` → `"channel_chat_id"`
   - Windows 文件系统不支持 `:` 作为文件名
   - 替换为 `_` 保持可读性

2. **安全文件名**: 使用 `safe_filename()` 函数
   ```python
   def safe_filename(name: str) -> str:
       """将字符串转换为安全文件名"""
       unsafe = '<>:"/\\|?*'
       for char in unsafe:
           name = name.replace(char, "_")
       return name.strip()
   ```

**路径示例**:
```
session_key = "telegram:123456789"
文件路径 = ~/.nanobot/sessions/telegram_123456789.jsonl
```

#### 4.2.2 get_or_create - 获取或创建会话

```python
def get_or_create(self, key: str) -> Session:
    """
    获取现有会话或创建新会话

    Args:
        key: 会话键（通常为 channel:chat_id）

    Returns:
        会话对象
    """
    # 检查缓存
    if key in self._cache:
        return self._cache[key]

    # 尝试从磁盘加载
    session = self._load(key)
    if session is None:
        session = Session(key=key)

    self._cache[key] = session
    return session
```

**三层查找策略**:

```
请求会话
    ↓
1. 检查内存缓存 ←── 是 → 返回（快速）
    ↓ 否
2. 检查磁盘文件 ←── 是 → 加载并缓存（中等）
    ↓ 否
3. 创建新会话 ←── 返回（慢）
```

**设计优势**:
- **性能优化**: 热会话在内存中，零 I/O 延迟
- **自动创建**: 不存在时自动创建新会话（无需手动初始化）
- **缓存一致性**: 加载后立即更新缓存

#### 4.2.3 _load - 从磁盘加载

```python
def _load(self, key: str) -> Session | None:
    """从磁盘加载会话"""
    path = self._get_session_path(key)

    if not path.exists():
        return None

    try:
        messages = []
        metadata = {}
        created_at = None

        with open(path) as f:
            for line in f:
                line = line.strip()
                if not line:
                    continue

                data = json.loads(line)

                if data.get("_type") == "metadata":
                    # 处理元数据行
                    metadata = data.get("metadata", {})
                    created_at = datetime.fromisoformat(data["created_at"]) if data.get("created_at") else None
                else:
                    # 处理消息行
                    messages.append(data)

        return Session(
            key=key,
            messages=messages,
            created_at=created_at or datetime.now(),
            metadata=metadata
        )
    except Exception as e:
        logger.warning(f"Failed to load session {key}: {e}")
        return None
```

**JSONL 格式解析**:

每个 JSONL 文件的格式：
```
第一行: {"_type": "metadata", "created_at": "...", "updated_at": "...", "metadata": {}}
后续行: {"role": "user", "content": "Hello", "timestamp": "..."}
下一行: {"role": "assistant", "content": "Hi there!", "timestamp": "..."}
...
```

**为什么使用 JSONL 而非 JSON 数组？**

| 特性 | JSON | JSONL |
|------|------|--------|
| 文件损坏 | 整个文件不可读 | 只影响损坏行 |
| 增量写入 | 需要重写整个文件 | 可追加新行 |
| 流式读取 | 需要解析整个文件 | 逐行读取 |
| 内存占用 | 全部加载 | 按需读取 |

#### 4.2.4 save - 保存到磁盘

```python
def save(self, session: Session) -> None:
    """保存会话到磁盘"""
    path = self._get_session_path(session.key)

    with open(path, "w") as f:
        # 先写入元数据
        metadata_line = {
            "_type": "metadata",
            "created_at": session.created_at.isoformat(),
            "updated_at": session.updated_at.isoformat(),
            "metadata": session.metadata
        }
        f.write(json.dumps(metadata_line) + "\n")

        # 写入消息
        for msg in session.messages:
            f.write(json.dumps(msg) + "\n")

    # 更新缓存
    self._cache[session.key] = session
```

**写入策略**:

1. **覆盖写入**: 使用 `"w"` 模式（而非追加）
   - 确保文件一致性
   - 每次保存都是完整的会话快照

2. **元数据优先**: 第一行是元数据
   - 快速读取基本信息
   - `list_sessions()` 可以只读第一行

3. **更新缓存**: 保存后更新内存缓存
   - 确保缓存与磁盘一致

#### 4.2.5 delete - 删除会话

```python
def delete(self, key: str) -> bool:
    """
    删除会话

    Args:
        key: 会话键

    Returns:
        删除成功返回 True，未找到返回 False
    """
    # 从缓存中移除
    self._cache.pop(key, None)

    # 删除文件
    path = self._get_session_path(key)
    if path.exists():
        path.unlink()  # 删除文件
        return True
    return False
```

#### 4.2.6 list_sessions - 列出所有会话

```python
def list_sessions(self) -> list[dict[str, Any]]:
    """
    列出所有会话

    Returns:
        会话信息字典列表
    """
    sessions = []

    for path in self.sessions_dir.glob("*.jsonl"):
        try:
            # 只读取元数据行（第一行）
            with open(path) as f:
                first_line = f.readline().strip()
                if first_line:
                    data = json.loads(first_line)
                    if data.get("_type") == "metadata":
                        sessions.append({
                            "key": path.stem.replace("_", ":"),
                            "created_at": data.get("created_at"),
                            "updated_at": data.get("updated_at"),
                            "path": str(path)
                        })
        except Exception:
            continue

    # 按更新时间降序排序
    return sorted(sessions, key=lambda x: x.get("updated_at", ""), reverse=True)
```

**关键优化**:

1. **只读第一行**: 不需要解析整个文件
   - 文件可能很大（上千条消息）
   - 只需元数据即可列出会话

2. **键恢复**: `path.stem.replace("_", ":")`
   - 文件名: `telegram_123456.jsonl`
   - 恢复键: `telegram:123456`

3. **排序**: 最近活跃的会话排前面
   - 使用 `reverse=True` 降序排列

---

## 数据流转

### 5.1 完整的会话生命周期

```
┌────────────────────────────────────────────────────────────┐
│                用户消息到达                             │
│  (Telegram/WhatsApp/Feishu/CLI)                   │
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│         InboundMessage 创建                          │
│  - channel: "telegram"                              │
│  - sender_id: "user_id"                            │
│  - chat_id: "123456789"                           │
│  - content: "Hello"                                 │
│  - session_key = "telegram:123456789" (自动计算)   │
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│         AgentLoop 处理消息                         │
│                                                     │
│  1. session = self.sessions.get_or_create(        │
│        msg.session_key)                                │
│     → 检查缓存 → 内存中返回                  │
│     → 未命中 → 尝试加载磁盘                      │
│     → 未找到 → 创建新 Session                    │
│                                                     │
│  2. messages = session.get_history(max=50)        │
│     → 获取最近50条消息                           │
│     → 转换为 LLM 格式                       │
│                                                     │
│  3. context.build_messages(                       │
│        history=messages,                                │
│        current_message=msg.content                         │
│     )                                                 │
│     → 构建完整上下文（系统提示词+历史+新消息）│
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│         LLM 处理                                 │
│  - 输入: 上下文消息 + 工具定义                    │
│  - 输出: LLMResponse                              │
│    * content: 文本响应                                │
│    * tool_calls: 工具调用（可选）                     │
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│         工具执行（如果有）                         │
│  - 执行工具函数                                    │
│  - 将结果添加到消息历史                          │
│  - 循环直到无工具调用                              │
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│         保存到会话                               │
│  session.add_message("user", msg.content)            │
│  session.add_message("assistant", final_content)       │
│  self.sessions.save(session)                          │
│     → 更新内存中的 messages 列表                  │
│     → 重写 JSONL 文件到磁盘                         │
└───────────────────┬────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│         发送响应                                │
│  OutboundMessage(                                    │
│      channel=msg.channel,                              │
│      chat_id=msg.chat_id,                              │
│      content=final_content                              │
│  )                                                   │
└─────────────────────────────────────────────────────────┘
```

### 5.2 多渠道会话隔离

```
会话键格式: "channel:chat_id"

┌──────────────────────────────────────────────────┐
│          ~/.nanobot/sessions/           │
├──────────────────────────────────────────────────┤
│ telegram_123456789.jsonl                │
│ telegram_987654321.jsonl                │
│ whatsapp_123456789.jsonl                │
│ feishu_abc123.jsonl                       │
│ cli_direct.jsonl                           │
└──────────────────────────────────────────────────┘

每个文件代表一个独立的对话上下文
- Telegram 用户 A 的对话独立于 Telegram 用户 B
- WhatsApp 用户独立于 Telegram 用户
- CLI 直接模式有独立的会话
```

**好处**:
- **隔离**: 不同用户/渠道互不干扰
- **持久化**: 重启后历史消息仍然保留
- **上下文**: LLM 可以引用之前的对话

---

## 设计模式总结

### 6.1 使用的设计模式

| 模式 | 位置 | 用途 |
|------|------|------|
| **数据传输对象 (DTO)** | `Session` 类 | 纯数据容器，无业务逻辑 |
| **管理器模式 (Manager)** | `SessionManager` 类 | 集中管理 Session 实例 |
| **缓存模式 (Cache)** | `self._cache` | 减少磁盘 I/O |
| **工厂方法 (Factory Method)** | `get_or_create()` | 按需创建 Session |
| **策略模式 (Strategy)** | JSONL vs JSON 存储 | 可选的存储策略 |
| **装饰器 (Decorator)** | `@dataclass` | 自动生成样板代码 |
| **模板方法 (Template Method)** | `_load()` 结构 | 定义加载流程，子类可重写 |

### 6.2 架构原则体现

1. **单一职责原则 (Single Responsibility)**:
   - `Session`: 只负责存储和访问会话数据
   - `SessionManager`: 只负责会话生命周期管理

2. **开闭原则 (Open/Closed)**:
   - 对扩展开放：可以添加新的存储后端（如数据库）
   - 对修改关闭：现有 SessionManager 无需修改

3. **里氏替换原则 (Liskov Substitution)**:
   - 任何 Session 实例都可以被互换使用
   - AgentLoop 不关心具体是哪个 Session

4. **接口隔离原则 (Interface Segregation)**:
   - Session 类只暴露必要方法：`add_message`, `get_history`, `clear`
   - 不暴露内部实现细节

5. **依赖倒置原则 (Dependency Inversion)**:
   - AgentLoop 依赖 SessionManager 接口（抽象）
   - 不关心 Session 如何存储（内存、磁盘、数据库）

---

## 学习要点

### 7.1 架构设计知识

#### 7.1.1 数据类 vs 普通类

**使用 dataclass 的优势**:

```python
# 普通类 - 需要手写大量代码
class SessionOld:
    def __init__(self, key, messages=None, ...):
        self.key = key
        self.messages = messages or []
        self.created_at = datetime.now()
        ...

    def __repr__(self):
        return f"Session(key={self.key})"

    def __eq__(self, other):
        if not isinstance(other, SessionOld):
            return False
        return self.key == other.key
    ... # 还需要 __hash__ 等

# dataclass - 自动生成
@dataclass
class SessionNew:
    key: str
    messages: list = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)
    # __init__, __repr__, __eq__, __hash__ 自动生成！
```

**何时使用 dataclass**:
- ✅ 纯数据容器（主要是字段，少方法）
- ✅ 需要序列化/反序列化
- ✅ 需要比较相等性

**何时使用普通类**:
- ✅ 有复杂的方法逻辑
- ✅ 需要封装实现细节
- ✅ 需要控制初始化过程

#### 7.1.2 内存缓存策略

**缓存 vs 无缓存对比**:

```python
# 无缓存 - 每次都读磁盘
def get_or_create_slow(self, key):
    return self._load(key) or Session(key=key)
    # 每次调用都触发 I/O

# 有缓存 - 快速路径
def get_or_create_fast(self, key):
    if key in self._cache:
        return self._cache[key]  # 零 I/O
    session = self._load(key)
    if session is None:
        session = Session(key=key)
    self._cache[key] = session  # 缓存结果
    return session
```

**缓存命中分析**:
```
假设场景：用户发送10条连续消息

无缓存: 10次磁盘读取 = ~500ms 延迟
有缓存: 1次磁盘读取 + 9次内存命中 = ~5ms 延迟

性能提升: 100倍！
```

**缓存一致性**:
- **写入时更新**: `save()` 方法更新缓存
- **手动清理**: 无自动失效机制（简单设计）
- **进程级**: 缓存在进程生命周期内

#### 7.1.3 JSONL vs JSON 存储选择

| 维度 | JSON 数组 | JSONL |
|------|----------|--------|
| **文件结构** | `[{}, {}, ...]` | `{}\n{}\n{}\n` |
| **增量写入** | 需要重写整个文件 | 可直接 `f.write(line + "\n")` |
| **增量读取** | 必须解析整个文件 | 可逐行 `for line in f` |
| **损坏隔离** | 一个错误影响整个文件 | 只影响单行 |
| **内存效率** | 需要加载全部到内存 | 流式处理 |
| **调试友好** | 需要格式化查看 | 可直接 `cat file` 查看 |
| **随机访问** | 可按索引读取 | 需要遍历 |

**为什么 nanobot 选择 JSONL**:
1. ✅ **增量追加**: 添加新消息不需要重写整个文件
2. ✅ **流式处理**: 可以逐行处理大文件，不需要全部加载到内存
3. ✅ **损坏隔离**: 单行损坏不影响其他行
4. ✅ **调试友好**: `cat` 直接可读

**何时应该用 JSON**:
- 需要频繁随机访问（按索引读取）
- 文件大小很小（几KB）
- 需要原子性（要么全部成功，要么全部失败）

#### 7.1.4 滑动窗口上下文管理

```python
def get_history(self, max_messages: int = 50) -> list[dict]:
    # 获取最近的 N 条消息
    recent = self.messages[-max_messages:] if len(self.messages) > max_messages else self.messages

    # 移除内部字段（timestamp 等），减少 token 消耗
    return [{"role": m["role"], "content": m["content"]} for m in recent]
```

**为什么需要限制历史？**

1. **LLM 上下文窗口限制**:
   - Claude 3.5: ~200K tokens
   - GPT-4: ~128K tokens
   - 超过限制会截断或报错

2. **成本控制**:
   - 更多历史 = 更多输入 token = 更高成本
   - 限制可以控制 API 费用

3. **响应速度**:
   - 更长上下文 = 更慢的推理时间

**滑动窗口策略**:
```
消息: [m1, m2, m3, m4, m5, m6, m7, m8, m9, m10]
窗口: max_messages=5

第1次请求: [m1, m2, m3, m4, m5]
第2次请求: [m1, m2, m3, m4, m5, m6, A: "Hi"]
第3次请求: [m2, m3, m4, m5, m6, m7, A: "Hi", R: "Hello"]
```

### 7.2 编程技巧

#### 7.2.1 类型注解的高级用法

```python
from typing import Any

def add_message(self, role: str, content: str, **kwargs: Any) -> None:
    # **kwargs 允许动态添加额外字段
    msg = {
        "role": role,
        "content": content,
        "timestamp": datetime.now().isoformat(),
        **kwargs  # 展开字典，添加额外键值对
    }
```

**使用场景**:
```python
# 基本用法
session.add_message("user", "Hello")
# → {"role": "user", "content": "Hello", "timestamp": "..."}

# 带额外字段
session.add_message("user", "Hello", source="telegram", user_id="123")
# → {"role": "user", "content": "Hello", "timestamp": "...", "source": "telegram", "user_id": "123"}
```

**好处**:
- 向前兼容：新功能不破坏现有代码
- 灵活性：调用者可以添加任何额外数据

#### 7.2.2 字典排序优化

```python
def list_sessions(self) -> list[dict]:
    sessions = [...]  # 收集会话

    # 按更新时间降序排序（最近的在前）
    return sorted(sessions, key=lambda x: x.get("updated_at", ""), reverse=True)
```

**排序技巧**:

```python
# 升序（从旧到新）
sorted(items, key=lambda x: x['date'])  # 最旧的在前

# 降序（从新到旧）
sorted(items, key=lambda x: x['date'], reverse=True)  # 最新的在前

# 多条件排序
sorted(items, key=lambda x: (x['priority'], x['date']))
# 优先级高的在前，同优先级的按日期排序

# 使用 .get() 避免缺失键报错
key=lambda x: x.get('updated_at', '')  # 缺失时使用空字符串
```

#### 7.2.3 文件路径安全处理

```python
def _get_session_path(self, key: str) -> Path:
    # 1. 替换冒号（Windows 文件系统不支持）
    safe_key = safe_filename(key.replace(":", "_"))

    # 2. 构建路径
    return self.sessions_dir / f"{safe_key}.jsonl"

def safe_filename(name: str) -> str:
    """将字符串转换为安全文件名"""
    unsafe = '<>:"/\\|?*'
    for char in unsafe:
        name = name.replace(char, "_")
    return name.strip()
```

**为什么需要安全处理**:

| 字符 | 问题系统 | 原因 |
|------|----------|------|
| `:` | Windows | 驱动器分隔符 (C:\) |
| `<` `>` `|` | Windows | 重定向操作符 |
| `?` `*` | Windows/Linux | 通配符 |
| `/` `\` | Windows/Linux | 路径分隔符 |
| `"` | Windows/Linux | 文件名包含引号问题 |

**实际转换示例**:
```
原始键: "telegram:123456?*测试>.txt"
转换后: "telegram_123456___测试_.txt"
```

#### 7.2.4 时间戳处理

```python
from datetime import datetime

# ISO 格式字符串
timestamp = datetime.now().isoformat()
# → "2026-02-09T10:30:45.123456"

# 从字符串解析
dt = datetime.fromisoformat("2026-02-09T10:30:45.123456")

# dataclass 默认工厂
@dataclass
class Session:
    created_at: datetime = field(default_factory=datetime.now)
    # ^^^^^^^^^^^^^^^^^^^
    # 每次创建新实例时，datetime.now() 会被调用
    # 每个实例有不同的时间戳
```

**时间序列化选择**:

| 格式 | 优点 | 缺点 |
|------|------|------|
| ISO 8601 | 标准、可读、可排序 | 较长 |
| Unix 时间戳 | 紧凑、易计算 | 不可读 |
| 自定义格式 | 灵活 | 需要文档 |

#### 7.2.5 错误处理策略

```python
def _load(self, key: str) -> Session | None:
    try:
        # 解析 JSONL 文件
        ...
    except Exception as e:
        logger.warning(f"Failed to load session {key}: {e}")
        return None  # 返回 None 而非抛出异常
```

**设计原则**:
- **记录但不崩溃**: 使用 `logger.warning` 记录错误
- **优雅降级**: 返回 `None` 表示加载失败
- **调用者处理**: `get_or_create()` 检查 `None` 并创建新会话

**错误处理对比**:

```python
# 糟糕的方式：向上传播异常
def bad_load(self, key):
    with open(path) as f:
        return json.loads(f.read())
    # 文件不存在 → FileNotFoundError 崩溃整个应用

# 好的方式：本地处理
def good_load(self, key):
    try:
        with open(path) as f:
            return json.loads(f.read())
    except FileNotFoundError:
        return None  # 文件不存在，返回空
    except json.JSONDecodeError:
        logger.warning(f"Corrupted file: {path}")
        return None  # 文件损坏，返回空
```

---

## 扩展指南

### 8.1 添加数据库后端

当前实现使用 JSONL 文件存储。如何改为数据库？

```python
import sqlite3
from typing import Optional
from dataclasses import asdict

class DatabaseSessionManager:
    """使用 SQLite 的会话管理器"""

    def __init__(self, db_path: Path):
        self.conn = sqlite3.connect(db_path)
        self._init_db()

    def _init_db(self):
        """初始化数据库表"""
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS sessions (
                key TEXT PRIMARY KEY,
                created_at TEXT,
                updated_at TEXT,
                metadata TEXT
            )
        """)
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS messages (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                session_key TEXT,
                role TEXT,
                content TEXT,
                timestamp TEXT,
                FOREIGN KEY (session_key) REFERENCES sessions(key)
            )
        """)

    def _load(self, key: str) -> Optional[Session]:
        """从数据库加载会话"""
        cursor = self.conn.execute("""
            SELECT s.key, s.created_at, s.updated_at, s.metadata,
                   m.role, m.content, m.timestamp
            FROM sessions s
            LEFT JOIN messages m ON s.key = m.session_key
            WHERE s.key = ?
            ORDER BY m.id ASC
        """, (key,))
        rows = cursor.fetchall()

        if not rows:
            return None

        # 第一行包含元数据
        metadata = json.loads(rows[0][3]) if rows[0][3] else {}
        created_at = datetime.fromisoformat(rows[0][1])

        # 构建消息列表
        messages = [
            {"role": row[4], "content": row[5], "timestamp": row[6]}
            for row in rows
        ]

        return Session(key=key, messages=messages, created_at=created_at, metadata=metadata)

    def save(self, session: Session) -> None:
        """保存会话到数据库"""
        # UPSERT 会话
        self.conn.execute("""
            INSERT OR REPLACE INTO sessions (key, created_at, updated_at, metadata)
            VALUES (?, ?, ?)
        """, (
            session.key,
            session.created_at.isoformat(),
            session.updated_at.isoformat(),
            json.dumps(session.metadata)
        ))

        # 删除旧消息
        self.conn.execute("DELETE FROM messages WHERE session_key = ?", (session.key,))

        # 插入新消息
        for msg in session.messages:
            self.conn.execute("""
                INSERT INTO messages (session_key, role, content, timestamp)
                VALUES (?, ?, ?)
            """, (session.key, msg["role"], msg["content"], msg.get("timestamp", "")))
```

**数据库优势**:
- ✅ **事务支持**: 原子性写入
- ✅ **索引查询**: 按时间、key 快速查找
- ✅ **随机访问**: 可直接查询特定消息
- ✅ **并发安全**: SQLite 支持多进程读
- ✅ **存储效率**: 二进制格式，比文本紧凑

### 8.2 添加会话过期机制

当前实现没有过期机制。如何添加？

```python
import time
from datetime import timedelta

class SessionManager:
    def __init__(self, workspace: Path, max_age_hours: int = 24 * 7):
        self.workspace = workspace
        self.max_age = timedelta(hours=max_age_hours)
        self.sessions_dir = ensure_dir(Path.home() / ".nanobot" / "sessions")
        self._cache: dict[str, Session] = {}

    def _load(self, key: str) -> Session | None:
        """从磁盘加载会话，检查过期"""
        path = self._get_session_path(key)

        if not path.exists():
            return None

        try:
            # 检查文件修改时间
            file_age = datetime.now() - datetime.fromtimestamp(path.stat().st_mtime)

            if file_age > self.max_age:
                logger.info(f"Session {key} expired (age: {file_age})")
                path.unlink()  # 删除过期会话
                return None

            # 加载会话内容
            ...
        except Exception as e:
            logger.warning(f"Failed to load session {key}: {e}")
            return None

    def cleanup_expired(self) -> int:
        """清理所有过期会话"""
        cleaned = 0
        now = datetime.now()

        for path in self.sessions_dir.glob("*.jsonl"):
            file_age = now - datetime.fromtimestamp(path.stat().st_mtime)
            if file_age > self.max_age:
                path.unlink()
                cleaned += 1
                logger.info(f"Cleaned up expired session: {path.stem}")

        return cleaned
```

**使用场景**:
```python
# 定期清理任务
import asyncio
from datetime import timedelta

async def periodic_cleanup(manager):
    while True:
        await asyncio.sleep(3600)  # 每小时检查一次
        cleaned = manager.cleanup_expired()
        if cleaned > 0:
            logger.info(f"Cleaned {cleaned} expired sessions")
```

### 8.3 添加会话搜索功能

```python
class SessionManager:
    def search_sessions(self, query: str, limit: int = 10) -> list[dict]:
        """
        搜索包含特定文本的会话

        Args:
            query: 搜索关键词
            limit: 最大返回数

        Returns:
            匹配的会话列表
        """
        results = []

        for path in self.sessions_dir.glob("*.jsonl"):
            try:
                # 扫描所有消息行
                with open(path, encoding='utf-8', errors='ignore') as f:
                    for i, line in enumerate(f):
                        if i == 0:  # 跳过元数据行
                            continue

                        data = json.loads(line)
                        content = data.get("content", "").lower()
                        if query.lower() in content:
                            # 找到匹配
                            results.append({
                                "key": path.stem.replace("_", ":"),
                                "path": str(path),
                                "matched_content": data.get("content", "")[:100],
                                "matched_role": data.get("role")
                            })
                            break  # 找到一个匹配就够了
            except Exception:
                continue

            if len(results) >= limit:
                break

        return results
```

**使用示例**:
```python
# 搜索讨论过特定主题的会话
sessions = manager.search_sessions("如何使用 API", limit=5)

for session in sessions:
    print(f"会话 {session['key']} 包含相关内容")
    print(f"  {session['matched_content']}")
```

### 8.4 添加会话统计功能

```python
@dataclass
class SessionStats:
    """会话统计信息"""
    total_sessions: int
    total_messages: int
    avg_messages_per_session: float
    oldest_session: str
    newest_session: str

class SessionManager:
    def get_statistics(self) -> SessionStats:
        """获取会话统计信息"""
        sessions = self.list_sessions()
        total_sessions = len(sessions)

        if total_sessions == 0:
            return SessionStats(
                total_sessions=0,
                total_messages=0,
                avg_messages_per_session=0,
                oldest_session="",
                newest_session=""
            )

        # 加载所有会话统计消息
        total_messages = 0
        oldest = sessions[-1]["created_at"]  # 已降序排列
        newest = sessions[0]["created_at"]

        for session_info in sessions:
            session = self._load(session_info["key"])
            if session:
                total_messages += len(session.messages)

        avg_messages = total_messages / total_sessions

        return SessionStats(
            total_sessions=total_sessions,
            total_messages=total_messages,
            avg_messages_per_session=round(avg_messages, 1),
            oldest_session=oldest,
            newest_session=newest
        )
```

### 8.5 添加会话导出/导入功能

```python
class SessionManager:
    def export_session(self, key: str, output_path: Path) -> bool:
        """导出会话为可读格式"""
        session = self._load(key)
        if not session:
            return False

        # 导出为 Markdown 格式
        with open(output_path, 'w', encoding='utf-8') as f:
            f.write(f"# 会话: {key}\n\n")
            f.write(f"创建时间: {session.created_at}\n\n")
            f.write("---\n\n")

            for msg in session.messages:
                role_icon = "👤" if msg["role"] == "user" else "🤖"
                f.write(f"{role_icon} **{msg['role']}**\n\n")
                f.write(f"{msg['content']}\n\n")
                f.write("---\n\n")

        return True

    def import_session(self, import_path: Path) -> bool:
        """从文件导入会话"""
        try:
            # 简单解析：每行 "role|content" 格式
            lines = import_path.read_text(encoding='utf-8').split('\n')
            messages = []
            key = import_path.stem

            for line in lines:
                if '|' not in line:
                    continue
                role, content = line.split('|', 1)
                messages.append({
                    "role": role.strip(),
                    "content": content.strip(),
                    "timestamp": datetime.now().isoformat()
                })

            session = Session(key=key, messages=messages)
            self.save(session)
            return True

        except Exception as e:
            logger.error(f"Failed to import session: {e}")
            return False
```

---

## 总结

### 核心架构特点

1. **轻量级设计**: 209 行代码实现完整会话管理
2. **双层架构**: 管理器 + 数据类，职责分离
3. **内存缓存**: 三层查找策略，性能优化
4. **JSONL 存储**: 流式处理，增量写入
5. **会话隔离**: 每个用户/渠道独立上下文
6. **滑动窗口**: 智能控制上下文长度

### 设计模式应用

- **数据传输对象 (DTO)**: Session 纯数据容器
- **管理器模式**: SessionManager 集中管理
- **缓存模式**: 内存字典减少 I/O
- **工厂方法**: 按需创建实例
- **策略模式**: 可替换存储后端

### 最佳实践

1. 使用 `@dataclass` 简化数据类定义
2. 实现内存缓存提升性能（三层查找）
3. 使用 JSONL 格式进行流式存储
4. 滑动窗口控制上下文大小
5. 安全的文件名处理（替换非法字符）
6. ISO 8601 时间戳格式
7. 优雅的错误处理（记录而不崩溃）
8. 类型注解提高代码质量

### 可学习的关键技术

- **如何设计会话管理系统**
- **如何实现内存缓存**
- **JSONL vs JSON 存储选择**
- **滑动窗口上下文管理**
- **类型注解高级用法**
- **文件路径安全处理**
- **数据类 vs 普通类选择**

### 实际 JSONL 文件示例

```jsonl
{"_type": "metadata", "created_at": "2026-02-05T13:40:59.840399", "updated_at": "2026-02-05T14:12:24.018696", "metadata": {}}
{"role": "user", "content": "Hello!", "timestamp": "2026-02-05T13:41:01.532963"}
{"role": "assistant", "content": "Hi there! How can I help you today?", "timestamp": "2026-02-05T13:41:01.532976"}
{"role": "user", "content": "What's the weather?", "timestamp": "2026-02-05T13:46:20.032526"}
{"role": "assistant", "content": "I don't have real-time weather data, but you can ask me to search for it!", "timestamp": "2026-02-05T13:46:20.032548"}
```

---

**报告生成时间**: 2026-02-09
**分析者**: AI Assistant
**目的**: 学习架构设计与编程知识
