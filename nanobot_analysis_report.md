# Nanobot 程序流程和结构分析报告

## 概述

Nanobot 是一个超轻量级的个人 AI 助手框架，核心代码约 4,000 行。程序采用模块化架构设计，主要基于 `nanobot/cli/commands.py` 作为 CLI 入口点，通过 Typer 框架实现命令行交互。

## 1. CLI 命令架构

### 1.1 CLI 结构

```
nanobot (主应用)
├── onboard          # 初始化配置和工作空间
├── agent            # 直接与 Agent 交互
│   ├── --message   # 单次消息模式
│   └── --session   # 会话 ID
├── gateway          # 启动网关服务器
│   ├── --port      # 端口 (默认 18790)
│   └── --verbose   # 详细输出
├── status          # 显示系统状态
├── channels        # 渠道管理子命令组
│   ├── status      # 显示渠道状态
│   └── login      # WhatsApp 设备链接 (QR 码)
└── cron           # 定时任务管理子命令组
    ├── list        # 列出定时任务
    ├── add         # 添加定时任务
    ├── remove      # 删除定时任务
    ├── enable      # 启用/禁用任务
    └── run         # 手动运行任务
```

### 1.2 CLI 入口流程图

```mermaid
flowchart TD
    A[用户执行 nanobot 命令] --> B[Typer 主应用]
    B --> C{选择命令类型}

    C --> D[onboard<br/>初始化配置]
    C --> E[agent<br/>直接对话]
    C --> F[gateway<br/>启动网关]
    C --> G[status<br/>查看状态]
    C --> H[channels<br/>渠道管理]
    C --> I[cron<br/>定时任务]

    D --> D1[创建配置文件]
    D1 --> D2[创建工作空间]
    D2 --> D3[创建模板文件]

    E --> E1{消息模式}
    E1 -->|单次消息| E2[process_direct]
    E1 -->|交互模式| E3[循环等待用户输入]
    E2 --> E4[返回响应]
    E3 --> E4

    F --> F1[加载配置]
    F1 --> F2[创建组件]
    F2 --> F3[启动异步服务]
    F3 --> F4[AgentLoop + Channels + Cron + Heartbeat]

    H --> H1[channels status<br/>显示渠道状态]
    H --> H2[channels login<br/>WhatsApp 设备链接]

    I --> I1[cron list/add/remove/enable/run<br/>定时任务管理]
```

## 2. 整体架构流程图

```mermaid
flowchart TB
    subgraph "CLI 层"
        CMD[Typer CLI 命令]
    end

    subgraph "配置层"
        CONFIG[Config Schema<br/>Pydantic 模型]
        LOADER[Config Loader<br/>加载/保存配置]
    end

    subgraph "消息总线层"
        BUS[MessageBus<br/>异步消息队列]
        INBOUND[Inbound 队列]
        OUTBOUND[Outbound 队列]
    end

    subgraph "Agent 核心层"
        LOOP[AgentLoop<br/>消息处理循环]
        PROVIDER[LLM Provider<br/>OpenRouter/Anthropic/OpenAI 等]
        TOOLS[Tool Registry<br/>工具注册表]
        CONTEXT[Context Builder<br/>上下文构建器]
        SESSIONS[Session Manager<br/>会话管理器]
        SUBAGENTS[Subagent Manager<br/>后台任务]
    end

    subgraph "工具层"
        FILE[文件系统工具]
        SHELL[Shell 执行工具]
        WEB[Web 搜索工具]
        MSG[消息工具]
        SPAWN[Subagent 工具]
        CRON_TOOL[Cron 工具]
    end

    subgraph "渠道层"
        CM[Channel Manager]
        TG[Telegram Channel]
        WA[WhatsApp Channel]
        FS[Feishu Channel]
    end

    subgraph "服务层"
        CRON_SVC[Cron Service<br/>定时任务]
        HB[Heartbeat Service<br/>心跳服务]
    end

    CMD --> CONFIG
    CMD --> LOADER
    CMD --> BUS
    CMD --> LOOP
    CMD --> CM
    CMD --> CRON_SVC
    CMD --> HB

    BUS --> INBOUND
    BUS --> OUTBOUND

    LOOP --> PROVIDER
    LOOP --> TOOLS
    LOOP --> CONTEXT
    LOOP --> SESSIONS
    LOOP --> SUBAGENTS
    LOOP --> BUS

    TOOLS --> FILE
    TOOLS --> SHELL
    TOOLS --> WEB
    TOOLS --> MSG
    TOOLS --> SPAWN
    TOOLS --> CRON_TOOL

    CM --> TG
    CM --> WA
    CM --> FS
    CM --> BUS

    CRON_SVC --> LOOP
    HB --> LOOP

    TG --> BUS
    WA --> BUS
    FS --> BUS
```

## 3. Gateway 启动流程详解

```mermaid
sequenceDiagram
    participant User as 用户
    participant CLI as nanobot gateway
    participant Config as 配置加载器
    participant Bus as MessageBus
    participant Agent as AgentLoop
    participant Channels as ChannelManager
    participant Cron as CronService
    participant Heartbeat as HeartbeatService
    participant Provider as LLMProvider
    participant Telegram as Telegram
    participant WhatsApp as WhatsApp
    participant Feishu as Feishu

    User->>CLI: 执行 nanobot gateway
    CLI->>Config: 加载配置
    Config-->>CLI: 返回 Config 对象
    CLI->>Bus: 创建 MessageBus
    CLI->>Provider: 创建 LiteLLMProvider
    CLI->>Cron: 创建 CronService
    CLI->>Agent: 创建 AgentLoop
    CLI->>Heartbeat: 创建 HeartbeatService
    CLI->>Channels: 创建 ChannelManager

    Channels->>Channels: 初始化渠道
    Channels->>Telegram: 启用?
    alt Telegram 启用
        Channels-->>Telegram: 创建实例
    end
    Channels->>WhatsApp: 启用?
    alt WhatsApp 启用
        Channels-->>WhatsApp: 创建实例
    end
    Channels->>Feishu: 启用?
    alt Feishu 启用
        Channels-->>Feishu: 创建实例
    end

    Note over CLI: 创建异步运行函数

    par 并行启动服务
        Cron->>Cron: start()
        Heartbeat->>Heartbeat: start()
        Agent->>Agent: run()
        Channels->>Channels: start_all()
    end

    loop Agent 主循环
        Agent->>Bus: consume_inbound()
        Note over Bus: 等待消息
        Bus-->>Agent: 返回 InboundMessage
        Agent->>Agent: _process_message()
        Agent->>Provider: chat()
        Provider-->>Agent: LLMResponse
        alt 有工具调用
            Agent->>Tools: execute()
            Tools-->>Agent: 执行结果
        end
        Agent->>Bus: publish_outbound()
    end

    loop 渠道监听循环
        Telegram->>Bus: publish_inbound()
        WhatsApp->>Bus: publish_inbound()
        Feishu->>Bus: publish_inbound()
    end

    loop Outbound 调度循环
        Channels->>Bus: consume_outbound()
        Bus-->>Channels: 返回 OutboundMessage
        alt 目标是 Telegram
            Channels->>Telegram: send()
        else 目标是 WhatsApp
            Channels->>WhatsApp: send()
        else 目标是 Feishu
            Channels->>Feishu: send()
        end
    end
```

## 4. Agent 消息处理流程详解

```mermaid
flowchart TD
    START([开始: 收到 InboundMessage]) --> CHECK{消息类型}

    CHECK -->|system 消息| SYSTEM[_process_system_message]
    CHECK -->|普通消息| NORMAL[_process_message]

    SYSTEM --> SYS_PARSE[解析原始 channel:chat_id]
    SYS_PARSE --> SYS_SESSION[获取/创建会话]
    SYS_SESSION --> SYS_CTX[更新工具上下文]
    SYS_CTX --> SYS_MSGS[构建消息列表]
    SYS_MSGS --> SYS_LOOP[Agent 循环]

    NORMAL --> SESSION[获取/创建会话<br/>通过 session_key]
    SESSION --> CONTEXT[更新工具上下文<br/>message, spawn, cron]
    CONTEXT --> BUILD[构建消息列表<br/>system + history + current]
    BUILD --> LOOP([Agent 循环])

    LOOP --> LLM_CALL[调用 LLM Provider]
    LLM_CALL --> LLM_RSP{响应类型}

    LLM_RSP -->|有工具调用| TOOL_CALLS[提取 tool_calls]
    LLM_RSP -->|无工具调用| FINAL[提取 content 作为最终响应]

    TOOL_CALLS --> ADD_ASSIST[添加 assistant 消息<br/>包含 tool_calls]
    ADD_ASSIST --> EXEC_TOOLS[遍历执行工具]

    EXEC_TOOLS --> TOOL[tools.execute]
    TOOL --> ADD_RESULT[添加 tool 消息<br/>包含执行结果]
    ADD_RESULT --> CHECK_ITER{迭代次数}

    CHECK_ITER -->|< max_iterations| LOOP
    CHECK_ITER -->|>= max_iterations| TIMEOUT[设置默认响应]

    FINAL --> SAVE[保存会话<br/>user + assistant 消息]
    TIMEOUT --> SAVE
    SAVE --> OUT[返回 OutboundMessage]
    SYS_LOOP --> SAVE

    OUT --> PUBLISH[发布到 outbound 队列]
    PUBLISH --> END([完成])
```

## 5. 消息总线 (MessageBus) 架构

```mermaid
flowchart LR
    subgraph "入站流向"
        TG[Telegram] -->|publish_inbound| IN_Q[Inbound 队列]
        WA[WhatsApp] -->|publish_inbound| IN_Q
        FS[Feishu] -->|publish_inbound| IN_Q
        SYS[System 消息] -->|publish_inbound| IN_Q
        CRON[Cron 服务] -->|publish_inbound| IN_Q
        CLI[CLI 直接调用] -->|process_direct| IN_Q
    end

    subgraph "消息总线"
        BUS[MessageBus]
        IN_Q
        OUT_Q[Outbound 队列]
    end

    subgraph "出站流向"
        OUT_Q -->|consume_outbound| DISP[调度器]
        DISP -->|分发回调| TG
        DISP -->|分发回调| WA
        DISP -->|分发回调| FS
    end

    BUS --> IN_Q
    BUS --> OUT_Q
    IN_Q --> BUS
    OUT_Q --> BUS
```

### 5.1 MessageBus 核心接口

| 方法 | 说明 | 方向 |
|------|------|------|
| `publish_inbound(msg)` | 发布入站消息 | Channel → Agent |
| `consume_inbound()` | 消费入站消息（阻塞） | Agent 读取 |
| `publish_outbound(msg)` | 发布出站消息 | Agent → Channel |
| `consume_outbound()` | 消费出站消息（阻塞） | Channel 读取 |
| `subscribe_outbound(channel, cb)` | 订阅特定渠道的出站消息 | Channel 注册 |
| `dispatch_outbound()` | 分发出站消息到订阅者 | 后台任务 |

## 6. 工具系统 (Tool Registry)

### 6.1 内置工具列表

| 工具名称 | 模块 | 功能 |
|----------|--------|------|
| `read_file` | filesystem.py | 读取文件内容 |
| `write_file` | filesystem.py | 写入文件 |
| `edit_file` | filesystem.py | 编辑文件 |
| `list_dir` | filesystem.py | 列出目录 |
| `exec` | shell.py | 执行 Shell 命令 |
| `web_search` | web.py | Web 搜索 |
| `web_fetch` | web.py | 获取网页内容 |
| `message` | message.py | 发送消息到渠道 |
| `spawn` | spawn.py | 创建后台子任务 |
| `cron` | cron.py | 管理定时任务 |

### 6.2 工具注册和执行流程

```mermaid
sequenceDiagram
    participant Loop as AgentLoop
    participant Registry as ToolRegistry
    participant Tool as 具体工具

    Note over Loop: _register_default_tools()
    Loop->>Registry: register(ReadFileTool())
    Loop->>Registry: register(WriteFileTool())
    Loop->>Registry: register(ExecTool())
    Loop->>Registry: register(WebSearchTool())
    Loop->>Registry: register(MessageTool())
    Loop->>Registry: register(SpawnTool())
    Loop->>Registry: register(CronTool())

    Note over Loop: LLM 返回工具调用
    Loop->>Registry: execute(tool_name, params)
    Registry->>Registry: validate_params(params)
    alt 参数验证失败
        Registry-->>Loop: 返回错误信息
    end

    Registry->>Tool: execute(**params)
    Tool-->>Registry: 返回执行结果
    Registry-->>Loop: 返回结果字符串
```

## 7. 上下文构建 (Context Builder)

```mermaid
flowchart TD
    START([build_messages]) --> SYS_Prompt[build_system_prompt]
    SYS_Prompt --> ID[_get_identity<br/>核心身份]
    ID --> BOOT[_load_bootstrap_files<br/>AGENTS.md, SOUL.md, USER.md]
    BOOT --> MEM[MemoryStore.get_memory_context<br/>MEMORY.md]
    MEM --> SKILLS1[加载 always 技能<br/>完整内容]
    SKILLS1 --> SKILLS2[构建技能摘要<br/>可用技能列表]

    ID --> MERGE1[合并为系统提示]
    BOOT --> MERGE1
    MEM --> MERGE1
    SKILLS1 --> MERGE1
    SKILLS2 --> MERGE1

    MERGE1 --> MSG_LIST[创建消息列表]
    MSG_LIST --> ADD_SYS[添加 system 消息]
    ADD_SYS --> ADD_HIST[添加历史消息]
    ADD_HIST --> ADD_USER[添加当前用户消息]
    ADD_USER --> MEDIA{有媒体文件?}
    MEDIA -->|是| ENCODE[Base64 编码图片]
    MEDIA -->|否| TEXT[纯文本]
    ENCODE --> ADD_USER
    TEXT --> ADD_USER

    ADD_USER --> RESULT([返回完整消息列表])
```

### 7.1 Bootstrap 文件加载顺序

1. **AGENTS.md** - Agent 指令和指南
2. **SOUL.md** - AI 个性特征
3. **USER.md** - 用户偏好设置
4. **TOOLS.md** - 工具文档（可选）
5. **IDENTITY.md** - 身份定义（可选）

### 7.2 技能加载机制

```mermaid
flowchart LR
    SKILLS_DIR[workspace/skills] --> SCANNER[SkillsLoader]

    SCANNER --> ALWAYS[扫描 always.md]
    ALWAYS --> LOAD_ALWAYS[加载完整内容到系统提示]

    SCANNER --> SKILL_MD[扫描每个技能的 SKILL.md]
    SKILL_MD --> SUMMARY[构建技能摘要列表]

    LOAD_ALWAYS --> SYSTEM[系统提示]
    SUMMARY --> SYSTEM

    AGENT[运行中] --> NEED{需要特定技能?}
    NEED -->|是| READ[read_file SKILL.md]
    READ --> AGENT
    NEED -->|否| AGENT
```

## 8. 会话管理 (Session Manager)

```mermaid
flowchart TD
    GET_OR_CREATE([get_or_create key]) --> CHECK_CACHE{缓存中存在?}

    CHECK_CACHE -->|是| RETURN_CACHE[返回缓存会话]
    CHECK_CACHE -->|否| CHECK_DISK{磁盘文件存在?}

    CHECK_DISK -->|是| LOAD[_load]
    CHECK_DISK -->|否| CREATE[创建新会话 Session]

    LOAD --> PARSE[解析 JSONL 文件]
    PARSE --> METADATA[读取元数据行]
    METADATA --> MESSAGES[读取所有消息行]
    MESSAGES --> BUILD[构建 Session 对象]

    BUILD --> CACHE[存入缓存]
    CREATE --> CACHE
    CACHE --> RETURN_CACHE

    ADD_MSG([add_message role content]) --> UPDATE[添加到 messages 列表]
    UPDATE --> TIME[更新 updated_at]

    GET_HISTORY([get_history max_messages]) --> RECENT[获取最近 N 条消息]
    RECENT --> FORMAT[转换为 LLM 格式]
    FORMAT --> RETURN_HIST[返回列表]

    SAVE([save session]) --> PATH[获取会话文件路径]
    PATH --> WRITE_META[写入元数据行]
    WRITE_META --> WRITE_MSGS[写入所有消息行]
    WRITE_MSGS --> UPDATE_CACHE[更新缓存]
```

### 8.1 会话文件格式 (JSONL)

```jsonl
{"_type":"metadata","created_at":"2026-02-10T10:00:00","updated_at":"2026-02-10T10:30:00","metadata":{}}
{"role":"user","content":"Hello","timestamp":"2026-02-10T10:00:00"}
{"role":"assistant","content":"Hi there!","timestamp":"2026-02-10T10:00:05"}
{"role":"user","content":"How are you?","timestamp":"2026-02-10T10:01:00"}
{"role":"assistant","content":"I'm doing well!","timestamp":"2026-02-10T10:01:05"}
```

## 9. 子代理系统 (Subagent Manager)

```mermaid
sequenceDiagram
    participant Main as 主 Agent
    participant SpawnTool as Spawn 工具
    participant SubMgr as SubagentManager
    participant Provider as LLMProvider
    participant Bus as MessageBus
    participant SubAgent as 子代理

    Main->>SpawnTool: spawn(task, label)
    SpawnTool->>SubMgr: spawn(task, label, origin)
    SubMgr->>SubMgr: 生成 task_id
    SubMgr->>SubMgr: 创建后台任务

    Note over SubMgr: 执行 _run_subagent

    SubMgr->>SubAgent: 创建隔离的 ToolRegistry
    SubAgent->>SubAgent: 构建子代理系统提示
    SubAgent->>Provider: chat()
    Provider-->>SubAgent: LLMResponse

    alt LLM 返回工具调用
        SubAgent->>SubAgent: execute(tool)
    else LLM 返回内容
        SubAgent->>SubAgent: 获取 final_result
    end

    SubAgent->>SubMgr: 任务完成
    SubMgr->>Bus: publish_inbound(system 消息)
    Bus-->>Main: 触发 _process_system_message

    Note over Main: 子代理结果作为系统消息处理
    Main->>Main: 生成用户友好的摘要
```

### 9.1 子代理工具限制

子代理只能使用以下工具（无消息和 spawn 功能）：
- 文件系统工具
- Shell 执行工具
- Web 搜索工具

这确保子代理专注于执行任务，不会：
- 发送消息给用户
- 创建更多子代理
- 访问主代理的对话历史

## 10. 定时任务服务 (Cron Service)

```mermaid
stateDiagram-v2
    [*] --> Stopped: 初始化
    Stopped --> Loading: start()
    Loading --> Loaded: _load_store
    Loaded --> Arming: _recompute_next_runs
    Arming --> Running: _arm_timer

    Running --> Waiting: 等待定时器
    Waiting --> Executing: 定时器触发
    Executing --> Running: _execute_job 完成

    Executing --> CheckNext: 执行后处理
    CheckNext --> Arming: 计算下次运行时间
    CheckNext --> Stopped: 一次性任务完成
    CheckNext --> Disabled: 任务禁用

    Running --> Stopping: stop()
    Stopping --> [*]: 停止
```

### 10.1 定时任务类型

| 类型 | 参数 | 说明 |
|------|------|------|
| `every` | `every_ms` | 每隔 N 毫秒执行一次 |
| `cron` | `expr` | Cron 表达式（如 `0 9 * * *`） |
| `at` | `at_ms` | 在指定时间戳执行一次 |

### 10.2 定时任务执行流程

```mermaid
flowchart TD
    START([定时器触发]) --> CHECK_DUE[检查到期的任务]
    CHECK_DUE --> DUE_LIST[获取所有 enabled 且 next_run_at 到期的任务]

    DUE_LIST --> EXEC_LOOP[遍历执行任务]
    EXEC_LOOP --> EXEC_ONE[_execute_job]

    EXEC_ONE --> CALLBACK[调用 on_job 回调<br/>通过 Agent]
    CALLBACK --> SUCCESS{执行成功?}
    SUCCESS -->|是| STATE_OK[state = ok]
    SUCCESS -->|否| STATE_ERR[state = error<br/>记录错误信息]

    STATE_OK --> UPDATE_TIME[更新 last_run_at_ms]
    STATE_ERR --> UPDATE_TIME

    UPDATE_TIME --> SCHED_TYPE{调度类型}
    SCHED_TYPE -->|every| CALC_NEXT[计算: now + every_ms]
    SCHED_TYPE -->|cron| CALC_CRON[使用 croniter 计算]
    SCHED_TYPE -->|at| ONE_SHOT[一次性任务]

    CALC_NEXT --> UPDATE_STATE[next_run_at_ms = 计算值]
    CALC_CRON --> UPDATE_STATE
    ONE_SHOT --> DELETE_FLAG{delete_after_run?}

    UPDATE_STATE --> SAVE[_save_store]
    SAVE --> ARM[_arm_timer]
    ARM --> WAIT_NEXT[等待下次触发]

    DELETE_FLAG -->|是| REMOVE_JOB[从 jobs 列表移除]
    DELETE_FLAG -->|否| DISABLE_JOB[enabled = False]

    REMOVE_JOB --> SAVE
    DISABLE_JOB --> SAVE
```

## 11. 渠道管理 (Channel Manager)

```mermaid
flowchart TD
    INIT([初始化 ChannelManager]) --> READ_CONFIG[读取配置]
    READ_CONFIG --> CHECK_TG{Telegram enabled?}
    CHECK_TG -->|是| CREATE_TG[创建 TelegramChannel]
    CHECK_TG -->|否| CHECK_WA

    CHECK_WA{WhatsApp enabled?} -->|是| CREATE_WA[创建 WhatsAppChannel]
    CHECK_WA -->|否| CHECK_FS
    CREATE_TG --> CHECK_WA

    CHECK_FS{Feishu enabled?} -->|是| CREATE_FS[创建 FeishuChannel]
    CHECK_FS -->|否| INIT_CHANNELS

    CREATE_WA --> CHECK_FS
    CREATE_FS --> INIT_CHANNELS[初始化 channels 字典]

    START([启动所有渠道]) --> START_DISP[启动出站调度器]
    START_DISP --> START_CHANS[启动每个渠道]

    START_CHANS --> LOOP[循环等待]
    LOOP --> STOP([停止所有渠道])

    DISPATCH([调度出站消息]) --> CONSUME[consume_outbound]
    CONSUME --> GET_CHAN{channels.get channel}
    GET_CHAN -->|找到| SEND[调用 channel.send]
    GET_CHAN -->|未找到| WARN[警告: 未知渠道]
    SEND --> CONTINUE[继续等待]
    WARN --> CONTINUE
```

### 11.1 各渠道特性

| 渠道 | 认证方式 | 连接类型 | 特殊功能 |
|------|----------|---------|---------|
| Telegram | Bot Token | HTTP Long Polling | 支持代理、语音转录（Groq） |
| WhatsApp | QR 扫码 | WebSocket | 需要 Node.js 桥接服务 |
| Feishu | App ID + Secret | WebSocket Long Connection | 无需公网 IP |

## 12. 配置系统

### 12.1 配置加载优先级

```mermaid
flowchart LR
    A[用户执行命令] --> B{配置文件存在?}

    B -->|是| C[load_config]
    B -->|否| D[使用默认 Config]

    C --> E[读取 JSON 文件]
    E --> F[convert_keys<br/>camelCase → snake_case]
    F --> G[Config.model_validate]
    G --> H[返回验证后的 Config]

    D --> H

    H --> USE[使用配置]
    SAVE[save_config] --> I[Config.model_dump]
    I --> J[convert_to_camel<br/>snake_case → camelCase]
    J --> K[写入 JSON 文件]
```

### 12.2 API Key 优先级

```mermaid
flowchart LR
    GET_KEY[get_api_key] --> OPR{OpenRouter?}
    OPR -->|有| USE_OP[使用 OpenRouter]
    OPR -->|无| DS{DeepSeek?}

    DS -->|有| USE_DS[使用 DeepSeek]
    DS -->|无| AN{Anthropic?}

    AN -->|有| USE_AN[使用 Anthropic]
    AN -->|无| OAI{OpenAI?}

    OAI -->|有| USE_OAI[使用 OpenAI]
    OAI -->|无| GM{Gemini?}

    GM -->|有| USE_GM[使用 Gemini]
    GM -->|无| ZP{Zhipu?}

    ZP -->|有| USE_ZP[使用 Zhipu]
    ZP -->|无| GQ{Groq?}

    GQ -->|有| USE_GQ[使用 Groq]
    GQ -->|无| VLL{vLLM?}

    VLL -->|有| USE_VLL[使用 vLLM]
    VLL -->|无| NONE[返回 None]
```

## 13. 文件目录结构

```
~/.nanobot/
├── config.json              # 主配置文件
├── workspace/              # 工作空间
│   ├── AGENTS.md          # Agent 指令
│   ├── SOUL.md            # AI 个性
│   ├── USER.md            # 用户信息
│   ├── memory/            # 长期记忆
│   │   └── MEMORY.md      # 持久化记忆
│   └── skills/           # 自定义技能
│       └── {skill-name}/
│           ├── SKILL.md   # 技能定义
│           └── always.md  # 是否始终加载
├── sessions/              # 会话存储
│   ├── cli_direct.jsonl   # CLI 会话
│   ├── telegram_123.jsonl  # Telegram 会话
│   └── whatsapp_456.jsonl # WhatsApp 会话
├── data/                 # 数据目录
│   └── cron/
│       └── jobs.json      # 定时任务存储
└── bridge/               # WhatsApp 桥接服务
    ├── package.json
    ├── dist/
    │   └── index.js
    └── ...
```

## 14. 关键数据流

### 14.1 用户通过 CLI 单次对话流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant CLI as nanobot agent -m
    participant Loop as AgentLoop
    participant Bus as MessageBus
    participant Provider as LLMProvider

    User->>CLI: nanobot agent -m "Hello"
    CLI->>Loop: process_direct("Hello")
    Loop->>Loop: 创建 InboundMessage
    Loop->>Loop: _process_message(msg)
    Loop->>Loop: 获取/创建会话
    Loop->>Loop: 构建消息列表
    Loop->>Provider: chat()
    Provider-->>Loop: LLMResponse
    Loop->>Loop: 保存会话
    Loop-->>CLI: 返回响应文本
    CLI-->>User: 🐈 响应内容
```

### 14.2 用户通过 Telegram 对话流程

```mermaid
sequenceDiagram
    participant User as Telegram 用户
    participant TG as Telegram Channel
    participant Bus as MessageBus
    participant Loop as AgentLoop
    participant Provider as LLMProvider
    participant CM as ChannelManager

    User->>TG: 发送消息
    TG->>TG: 接收并验证
    alt 用户在允许列表
        TG->>Bus: publish_inbound(InboundMessage)
        Bus->>Loop: consume_inbound()
        Loop->>Loop: _process_message(msg)
        Loop->>Loop: 获取/创建会话
        Loop->>Loop: 构建消息列表
        Loop->>Provider: chat()
        Provider-->>Loop: LLMResponse
        Loop->>Bus: publish_outbound(OutboundMessage)
        Bus->>CM: consume_outbound()
        CM->>TG: send(msg)
        TG->>User: 发送响应
    else 用户不在允许列表
        TG->>User: 忽略/拒绝
    end
```

### 14.3 定时任务触发流程

```mermaid
sequenceDiagram
    participant Cron as CronService
    participant Timer as 定时器
    participant Loop as AgentLoop
    participant Provider as LLMProvider
    participant Bus as MessageBus
    participant CM as ChannelManager

    Note over Cron: 定时器等待
    Timer->>Cron: 触发 (时间到)
    Cron->>Cron: _on_timer()
    Cron->>Cron: 获取到期任务

    loop 每个到期任务
        Cron->>Loop: process_direct(message, session_key)
        Loop->>Loop: _process_message(msg)
        Loop->>Provider: chat()
        Provider-->>Loop: LLMResponse
        Loop-->>Cron: 返回响应

        alt deliver=true 且配置了 to
            Cron->>Bus: publish_outbound(OutboundMessage)
            Bus->>CM: consume_outbound()
            CM->>Channel: send(msg)
            Channel->>User: 发送消息
        end

        Cron->>Cron: 更新任务状态
        Cron->>Cron: 计算下次运行时间
    end

    Cron->>Cron: _arm_timer()
```

## 15. 异常处理和错误恢复

```mermaid
flowchart TD
    ERROR([发生异常]) --> CATCH{异常类型}

    CATCH -->|配置加载失败| CONFIG_ERR[使用默认配置]
    CONFIG_ERR --> LOG_WARN[记录警告日志]
    LOG_WARN --> CONTINUE[继续执行]

    CATCH -->|消息处理失败| MSG_ERR[捕获异常]
    MSG_ERR --> LOG_ERROR[记录错误日志]
    LOG_ERROR --> ERR_MSG[发送错误响应]
    ERR_MSG --> BUS_PUB[publish_outbound]
    BUS_PUB --> CONTINUE

    CATCH -->|工具执行失败| TOOL_ERR[返回错误字符串]
    TOOL_ERR --> ADD_MSG[添加 tool 消息到历史]
    ADD_MSG --> RETRY[重试 LLM]

    CATCH -->|渠道发送失败| CH_ERR[捕获异常]
    CH_ERR --> LOG_CH[记录渠道错误]
    LOG_CH --> SKIP_MSG[跳过当前消息]
    SKIP_MSG --> CONTINUE
```

## 总结

Nanobot 的核心架构设计遵循以下原则：

1. **解耦设计**：消息总线实现渠道和 Agent 的完全解耦
2. **异步优先**：所有 I/O 操作都使用 asyncio 异步处理
3. **可扩展性**：通过 Tool Registry 和 Channel Manager 实现插件式扩展
4. **状态持久化**：会话、配置、定时任务都持久化到磁盘
5. **轻量级**：约 4,000 行核心代码，易于理解和修改

通过 CLI 命令 `nanobot gateway` 启动后，系统形成一个完整的事件循环：
- 渠道接收用户消息 → MessageBus 入站队列 → AgentLoop 处理 → LLM 推理 → 工具执行 → MessageBus 出站队列 → 渠道发送响应
- 同时，CronService 和 HeartbeatService 可以触发内部消息进入处理流程
- 复杂任务可以交给子代理后台执行，完成后通过系统消息通知主代理
