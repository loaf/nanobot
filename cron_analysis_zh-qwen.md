# nanobot 定时任务模块分析报告

## 概述

nanobot 的定时任务（cron）模块提供了一个轻量级的调度系统，用于在指定时间执行AI代理任务。该模块允许用户安排一次性、周期性或基于cron表达式的任务，并与AI代理无缝集成。

## 目录结构

```
nanobot/
├── cron/
│   ├── __init__.py     # 导入CronService, CronJob, CronSchedule
│   ├── service.py      # 核心调度服务实现
│   └── types.py        # 类型定义
```

## 核心组件分析

### 1. 类型定义 (types.py)

定义了任务调度所需的核心数据类型：

- **CronSchedule**: 调度配置
  - `kind`: 调度类型 ("at", "every", "cron")
  - `at_ms`: 一次性执行的时间戳（毫秒）
  - `every_ms`: 周期执行间隔（毫秒）
  - `expr`: Cron表达式（如 "0 9 * * *"）
  - `tz`: 时区

- **CronPayload**: 执行载荷
  - `kind`: 执行类型 ("agent_turn", "system_event")
  - `message`: 传递给AI代理的消息
  - `deliver`: 是否将响应发送回渠道
  - `channel`: 目标渠道 (e.g., "whatsapp", "telegram")
  - `to`: 接收者标识

- **CronJobState**: 运行状态
  - `next_run_at_ms`: 下次执行时间
  - `last_run_at_ms`: 最后执行时间
  - `last_status`: 最后执行状态 ("ok", "error", "skipped")
  - `last_error`: 错误信息

- **CronJob**: 完整的任务定义
  - 包含ID、名称、状态、调度配置、载荷和创建时间等

### 2. 调度服务 (service.py)

CronService 是核心调度器类，主要功能包括：

#### 初始化
- 接收存储路径和回调函数
- 管理任务持久化和执行

#### 数据持久化
- `_load_store()`: 从磁盘加载任务配置
- `_save_store()`: 将任务配置保存到磁盘
- 使用JSON格式存储任务列表

#### 调度逻辑
- `_compute_next_run()`: 计算下次执行时间
  - "at": 一次性执行
  - "every": 基于间隔重复执行
  - "cron": 基于cron表达式执行（使用croniter库）

#### 定时器机制
- `_arm_timer()`: 设置下一个定时器触发
- `_on_timer()`: 处理定时器触发事件
- `_execute_job()`: 执行单个任务

#### 公共API
- `list_jobs()`: 列出所有任务
- `add_job()`: 添加新任务
- `remove_job()`: 删除任务
- `enable_job()`: 启用/禁用任务
- `run_job()`: 手动执行任务
- `status()`: 获取服务状态

### 3. 工具集成 (agent/tools/cron.py)

CronTool 提供了在AI对话中使用定时任务的功能：

- 支持 `add`, `list`, `remove` 三种操作
- 可以设置消息、执行间隔或cron表达式
- 自动与当前会话上下文绑定

### 4. CLI 集成 (cli/commands.py)

提供了命令行接口来管理定时任务：

- `nanobot cron list`: 列出所有任务
- `nanobot cron add`: 添加新任务
- `nanobot cron remove`: 删除任务
- `nanobot cron enable/disable`: 启用/禁用任务
- `nanobot cron run`: 手动执行任务

参数示例：
- `--message`: 任务消息
- `--every`: 执行间隔（秒）
- `--cron`: Cron表达式
- `--at`: 一次性执行时间

## 运行逻辑分析

### 1. 启动流程
1. Gateway启动时创建CronService实例
2. 加载已存任务配置
3. 计算所有任务的下一次执行时间
4. 设置定时器等待最近的任务

### 2. 执行流程
1. 定时器触发，检查当前时间是否有到期任务
2. 并行执行所有到期任务
3. 更新任务状态（上次执行时间、下次执行时间等）
4. 保存状态并重新计算下一次唤醒时间

### 3. 任务类型处理
- **一次性任务** (`kind="at"`):
  - 执行后自动禁用或删除（根据delete_after_run）
- **周期任务** (`kind="every"`):
  - 每次执行后计算下一次执行时间
- **Cron任务** (`kind="cron"`):
  - 使用cron表达式计算下一次执行时间

## 编辑技巧

### 1. 代码扩展建议
- 在添加新的调度类型时，需要修改CronSchedule的kind枚举
- 增加新的执行动作需扩展CronPayload的kind类型
- 新增状态字段需要同步更新序列化/反序列化逻辑

### 2. 调试技巧
- 启动时检查 `~/.nanobot/data/cron/jobs.json` 文件确认任务配置
- 查看日志中的"Cron:"前缀信息跟踪任务执行
- 使用 `nanobot cron list` 查看当前任务状态

### 3. 性能优化
- 任务状态计算采用批处理方式减少IO操作
- 使用内存缓存避免重复解析任务配置
- 定时器机制使用最小堆算法选择下一个唤醒时机

### 4. 错误处理
- 任务执行失败会记录错误状态但不影响其他任务
- Cron表达式解析失败会被捕获并返回None
- 磁盘读写异常有适当的降级策略

## 使用场景

### 1. 日常提醒
- 每天定时发送问候消息
- 周期性检查系统状态

### 2. 自动化任务
- 定时获取天气信息
- 定期总结工作进度

### 3. 长期规划
- 重要事件提醒
- 定期知识回顾

## 总结

nanobot的cron模块设计简洁高效，通过异步定时器实现了灵活的调度机制。它与AI代理深度集成，使得用户可以通过自然语言方便地创建和管理定时任务。整个系统具有良好的可扩展性和稳定性，适合轻量级AI助手的各种自动化需求。