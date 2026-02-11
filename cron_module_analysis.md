# nanobot Cron 模块分析报告

## 1. 概述

`nanobot` 的 Cron 模块提供了一个轻量级的异步任务调度系统，旨在让 AI Agent 能够在指定时间或周期性地主动执行任务。它不仅支持标准的 Cron 表达式，还支持一次性任务 (`at`) 和简单周期任务 (`every`)。该模块与 `nanobot` 的核心组件（如 `AgentLoop` 和 `MessageBus`）紧密集成，实现了从定时触发到 Agent 执行再到结果推送的完整闭环。

## 2. 目录结构与核心文件

`nanobot/cron` 目录包含以下核心文件：

- [nanobot/cron/\_\_init\_\_.py](file:///d:/2/nanobot/nanobot/cron/__init__.py): 模块入口，导出核心类。
- [nanobot/cron/types.py](file:///d:/2/nanobot/nanobot/cron/types.py): 定义数据模型（Data Classes）。
- [nanobot/cron/service.py](file:///d:/2/nanobot/nanobot/cron/service.py): 核心服务实现，负责调度逻辑、持久化和执行。

## 3. 核心实现分析

### 3.1 数据模型 (types.py)

该文件使用 Python 的 `dataclasses` 定义了调度系统的核心数据结构。

*   **CronSchedule**: 定义任务的调度规则。
    *   `kind`: 调度类型，支持 `"at"` (指定时间), `"every"` (间隔时间), `"cron"` (Cron 表达式)。
    *   对应字段: `at_ms`, `every_ms`, `expr` (Cron 表达式), `tz` (时区)。
    *   [代码引用: types.py:8-18](file:///d:/2/nanobot/nanobot/cron/types.py#L8-L18)

*   **CronPayload**: 定义任务触发时要执行的内容。
    *   `kind`: 通常为 `"agent_turn"` (Agent 对话轮次)。
    *   `message`: 发送给 Agent 的提示词 (Prompt)。
    *   `deliver`: 是否将 Agent 的响应发送到外部渠道。
    *   `channel`, `to`: 指定响应发送的目标渠道和接收人。
    *   [代码引用: types.py:22-30](file:///d:/2/nanobot/nanobot/cron/types.py#L22-L30)

*   **CronJob**: 表示一个完整的定时任务。
    *   包含 `id`, `name`, `enabled`, `schedule`, `payload` 以及运行时状态 `state`。
    *   [代码引用: types.py:42-53](file:///d:/2/nanobot/nanobot/cron/types.py#L42-L53)

### 3.2 调度服务 (service.py)

`CronService` 是整个模块的大脑，负责管理任务生命周期和调度执行。

#### 3.2.1 初始化与持久化
*   `CronService` 初始化时接收一个 `store_path` (通常是 `~/.nanobot/data/cron/jobs.json`) 和一个 `on_job` 回调函数。
*   **加载**: `_load_store` 方法从 JSON 文件读取任务列表。
*   **保存**: `_save_store` 方法将任务列表写回 JSON 文件，确保任务配置持久化。
*   [代码引用: service.py:45-146](file:///d:/2/nanobot/nanobot/cron/service.py#L45-L146)

#### 3.2.2 调度计算
*   `_compute_next_run` 函数根据调度类型计算下一次执行的时间戳。
    *   对于 `cron` 类型，它使用 `croniter` 库解析表达式。
    *   对于 `every` 类型，它基于当前时间加上间隔。
    *   [代码引用: service.py:19-39](file:///d:/2/nanobot/nanobot/cron/service.py#L19-L39)

#### 3.2.3 运行循环 (Event Loop Integration)
`CronService` 不使用独立的线程，而是利用 `asyncio` 的事件循环。

*   **启动**: `start()` 方法加载任务，重算下次运行时间，并启动定时器。
    *   [代码引用: service.py:147-155](file:///d:/2/nanobot/nanobot/cron/service.py#L147-L155)
*   **定时器**: `_arm_timer()` 计算所有启用任务中最近的一个执行时间 (`_get_next_wake_ms`)，然后创建一个 `asyncio.create_task` 在该时刻唤醒。
    *   [代码引用: service.py:180-198](file:///d:/2/nanobot/nanobot/cron/service.py#L180-L198)
*   **触发**: 当定时器到期，`_on_timer()` 被调用，它会找出所有已到期的任务并执行它们，然后再次调用 `_arm_timer()` 进入下一轮等待。
    *   [代码引用: service.py:199-215](file:///d:/2/nanobot/nanobot/cron/service.py#L199-L215)

#### 3.2.4 任务执行
*   `_execute_job` 方法负责执行单个任务。
*   它调用 `self.on_job(job)` 回调函数，这是 Cron 模块与外部系统（Agent）交互的接口。
*   执行后，它会更新任务状态 (`last_run_at_ms`, `last_status`) 并计算下一次运行时间。
*   [代码引用: service.py:216-248](file:///d:/2/nanobot/nanobot/cron/service.py#L216-L248)

## 4. 运行过程与集成

Cron 模块主要通过 CLI 命令进行管理，并在 Gateway 模式下运行。

### 4.1 CLI 管理
在 [nanobot/cli/commands.py](file:///d:/2/nanobot/nanobot/cli/commands.py) 中，`cron_app` 定义了 `add`, `list`, `remove`, `enable`, `run` 等命令。这些命令实例化 `CronService`，直接操作底层的 JSON 存储文件，无需启动后台服务即可修改配置。

### 4.2 Gateway 运行流程
当用户运行 `nanobot gateway` 时，Cron 服务被启动并集成到主循环中：

1.  **实例化**: 在 `gateway` 函数中创建 `CronService`。
    *   [代码引用: cli/commands.py:187-188](file:///d:/2/nanobot/nanobot/cli/commands.py#L187-L188)

2.  **回调绑定**: 定义 `on_cron_job` 回调函数。当任务触发时：
    *   调用 `agent.process_direct` 将任务消息 (`payload.message`) 发送给 Agent。
    *   如果配置了 `deliver=True`，则将 Agent 的响应封装为 `OutboundMessage`，通过 `MessageBus` 发送到指定渠道（如 Telegram/WhatsApp）。
    *   [代码引用: cli/commands.py:202-216](file:///d:/2/nanobot/nanobot/cli/commands.py#L202-L216)

3.  **启动**: `await cron.start()` 被调用，开始监听时间事件。
    *   [代码引用: cli/commands.py:261](file:///d:/2/nanobot/nanobot/cli/commands.py#L261)

## 5. 总结

nanobot 的 Cron 模块实现简洁而高效：
*   **低耦合**: 核心逻辑不依赖具体的 Agent 实现，仅通过回调交互。
*   **持久化**: 基于 JSON 文件，易于查看和备份。
*   **异步原生**: 完美融入 Python 的 `asyncio` 生态，资源占用极低。
*   **灵活**: 支持一次性、周期性和 Cron 表达式，满足多种自动化需求。
