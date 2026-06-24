---
id: "analysis-hermes-datamodels-005"
title: "状态持久化与数据模型分析"
aliases: ["hermes data models", "SessionDB", "SQLite persistence"]
type: "analysis"
category: "design/analysis/hermes"
tags: ["hermes-agent", "data-model", "SQLite", "FTS5", "persistence"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: "analysis-hermes-overview-001"
related_docs:
  - id: "analysis-hermes-overview-001"
    relation: "parent"
    path: "./01-overview.md"
  - id: "analysis-hermes-runtime-002"
    relation: "related_to"
    path: "./02-agent-runtime.md"
---

<!-- @section: overview -->
# 状态持久化与数据模型分析

## 系统概述

Hermes Agent 使用 **SQLite** 作为主要持久化引擎，采用 WAL 模式和 FTS5 全文搜索。与 new-api 不同，Hermes 不使用 Redis 缓存，所有状态直接读写本地 SQLite。

## 一、SessionDB — 核心数据库

### 数据库文件

| 数据库 | 路径 | 用途 |
|--------|------|------|
| SessionDB | `~/.hermes/state.db` | 会话、消息、全文搜索 |

### 数据库特性

- **引擎**: SQLite
- **模式**: WAL（Write-Ahead Logging）— 支持并发读写
- **全文搜索**: FTS5（两个虚拟表，分别针对拉丁文本和 CJK）
- **线程安全**: `check_same_thread=False` + 显式事务控制
- **模式版本**: 11（自动迁移）

### 表结构

`hermes_state.py::SCHEMA_SQL` 共定义 6 张表（4 张普通表 + 2 张 FTS5 虚拟表）：`schema_version` / `sessions` / `messages` / `state_meta` / `messages_fts` / `messages_fts_trigram`。

#### schema_version 表

```sql
CREATE TABLE schema_version (
    version INTEGER NOT NULL
);
```

启动时读取 `version` 列与代码中的 `SCHEMA_VERSION = 11` 比对，缺列则按 "Beets/sqlite-utils 模式"（对比期望 SCHEMA_SQL 与实时模式，添加缺失列）原地迁移并把版本写回此表。和下面的通用键值 `state_meta` 不要混淆。

#### sessions 表

```sql
CREATE TABLE sessions (
    id              TEXT PRIMARY KEY,
    source          TEXT,          -- 来源: cli/gateway/telegram/discord...
    user_id         TEXT,          -- 用户标识
    model           TEXT,          -- 模型名
    model_config    TEXT,          -- JSON 模型配置
    system_prompt   TEXT,          -- 系统提示
    parent_session_id TEXT,       -- 父会话 ID（压缩拆分）
    started_at      INTEGER,       -- 开始时间戳
    ended_at        INTEGER,       -- 结束时间戳
    end_reason      TEXT,          -- 结束原因
    message_count   INTEGER,       -- 消息数
    tool_call_count INTEGER,       -- 工具调用次数
    input_tokens    INTEGER,       -- 输入 Token 总数
    output_tokens   INTEGER,       -- 输出 Token 总数
    cache_tokens    INTEGER,       -- 缓存 Token 数
    reasoning_tokens INTEGER,      -- 推理 Token 数
    billing_details TEXT,          -- JSON 计费详情
    estimated_cost  REAL,          -- 预估成本
    actual_cost     REAL,          -- 实际成本
    title           TEXT,          -- 会话标题
    api_call_count  INTEGER        -- API 调用次数
);
```

**关键设计**:
- `parent_session_id` 支持会话拆分链（上下文压缩）
- `billing_details` 存储 JSON 格式的多提供商计费信息
- `estimated_cost` vs `actual_cost` 支持成本预估对比

#### messages 表

```sql
CREATE TABLE messages (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id      TEXT NOT NULL,
    role            TEXT NOT NULL,       -- system/user/assistant/tool
    content         TEXT,                -- 消息内容
    tool_call_id    TEXT,                -- 工具调用 ID
    tool_calls      TEXT,                -- JSON 工具调用列表
    tool_name       TEXT,                -- 工具名称
    timestamp       INTEGER,             -- 时间戳
    token_count     INTEGER,             -- Token 数
    finish_reason   TEXT,                -- 完成原因
    reasoning       TEXT,                -- 推理内容
    reasoning_content TEXT,              -- 原始推理内容
    reasoning_details TEXT,              -- JSON 推理详情
    codex_reasoning_items TEXT,         -- Codex 推理项
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);
```

**关键设计**:
- 支持多种推理格式（Anthropic thinking, OpenAI reasoning, Codex reasoning）
- `tool_calls` 以 JSON 存储完整的工具调用信息
- `token_count` 记录每条消息的 Token 消耗

#### state_meta 表

```sql
CREATE TABLE state_meta (
    key   TEXT PRIMARY KEY,
    value TEXT
);
```

通用键值元数据存储，用于：
- 迁移状态标记（"已完成 N 次旧版 JSONL 会话搬运"等）
- 全局计数器
- 其他需要持久化的轻量配置

> 模式版本号本身存放于上面的 `schema_version` 表，不在 `state_meta` 中。

#### messages_fts 表（拉丁文本搜索）

```sql
CREATE VIRTUAL TABLE messages_fts USING fts5(
    content,
    tokenize='unicode61'
);
```

适用于英文等拉丁语言的全文本搜索。

#### messages_fts_trigram 表（CJK 搜索）

```sql
CREATE VIRTUAL TABLE messages_fts_trigram USING fts5(
    content,
    tokenize='trigram'
);
```

适用于中文、日文、韩文的子字符串搜索。

### 写入策略

- **显式事务**: `BEGIN IMMEDIATE` 开始写入事务
- **重试逻辑**: 最多 15 次重试，随机抖动（20-150ms）避免多进程护航效应
- **WAL 检查点**: 每 50 次写入执行一次，控制 WAL 文件大小
- **自动迁移**: 启动时对比实时模式与期望 SCHEMA_SQL，添加缺失列
- **旧格式迁移**: 从基于 JSONL 的旧会话存储自动迁移

## 二、其他持久化存储

### 配置文件

| 文件 | 格式 | 用途 |
|------|------|------|
| `~/.hermes/config.yaml` | YAML | 主配置文件 |
| `~/.hermes/.env` | 键值文件 | API 密钥和环境变量 |
| `~/.hermes/SOUL.md` | Markdown | 自定义 Agent 角色 |

### 技能存储

| 路径 | 格式 | 用途 |
|------|------|------|
| `~/.hermes/skills/` | 目录 + SKILL.md | 已安装技能 |
| `skills/` (仓库内) | 目录 + SKILL.md | 捆绑技能（清单同步） |

**同步策略**: `skills_sync.py` 基于清单同步：
- 新技能 → 复制
- 用户修改的技能 → 跳过
- 用户删除的技能 → 尊重

### 记忆存储

| 文件 | 用途 |
|------|------|
| `~/.hermes/memories/MEMORY.md` | Agent 长期记忆 |
| `~/.hermes/memories/USER.md` | 用户偏好记忆 |

### 定时任务存储

`~/.hermes/cron/jobs.json`:
- JSON 数组，每个作业一个对象
- 原子替换写入（write-temp + rename）
- 输出: `~/.hermes/cron/output/{job_id}/{timestamp}.md`

### 缓存和临时文件

| 路径 | 用途 |
|------|------|
| `~/.hermes/channel_directory.json` | 消息通道目录缓存 |
| `~/.hermes/plans/` | Agent 计划 (*.md) |
| `~/.hermes/skins/` | 用户自定义 UI 皮肤 |
| `~/.hermes/logs/` | 应用日志（agent.log, errors.log, gateway.log） |
| `~/.hermes/sessions/` | 遗留 JSONL 会话（正在迁移到 SQLite） |

## 三、日志系统

### 日志文件

`hermes_logging.py` — 集中日志设置：

| 文件 | 级别 | 用途 |
|------|------|------|
| `agent.log` | INFO+ | 所有 Agent/工具/会话活动 |
| `errors.log` | WARNING+ | 仅错误（快速分类） |
| `gateway.log` | INFO+ | 仅网关事件（`mode="gateway"` 时） |

### 日志特性

- **轮转**: `RotatingFileHandler`，可配置最大大小（默认 5MB）和备份数（默认 3）
- **脱敏**: `RedactingFormatter` — PII/密钥删除（`agent/redact.py`）
- **组件过滤**: `_ComponentFilter` — 路由网关特定日志
- **会话关联**: `_install_session_record_factory()` — 注入 `%(session_tag)s` 格式字段
- **线程安全**: `threading.local()` 会话上下文
- **噪音抑制**: openai, httpx, httpcore, asyncio, grpc 等第三方库设为 WARNING

## 四、配置系统

### 配置层级

```
环境变量 (.env)
    ↓ 覆盖
config.yaml (用户配置)
    ↓ 合并
DEFAULT_CONFIG (代码默认值)
```

### 配置结构 (config.yaml)

```yaml
# 模型配置
model: "anthropic/claude-sonnet-4"
provider: "anthropic"
base_url: ~
fallback_models: []

# API 密钥 (推荐用 .env)
api_keys:
  openai: "${OPENAI_API_KEY}"

# 工具配置
toolsets: ["default"]
yolo: false              # 自动批准模式

# 记忆
memory:
  provider: ~            # 外部记忆提供者

# 上下文
context:
  engine: "compressor"   # 压缩引擎

# 日志
logging:
  max_log_size_mb: 5
  backup_count: 3

# MCP 服务器
mcp_servers:
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]

# 网关
gateway:
  platforms:
    telegram:
      enabled: true
      token: "${TELEGRAM_BOT_TOKEN}"

# 定时任务
cron:
  jobs: []
```

## 五、轨迹数据

### 格式

`agent/trajectory.py` — ShareGPT 格式 JSONL：

```json
{
  "conversations": [
    {"from": "human", "value": "用户输入"},
    {"from": "gpt", "value": "助手响应"},
    {"from": "tool", "value": "工具结果"}
  ],
  "system": "系统提示",
  "model": "模型名"
}
```

### 压缩

`trajectory_compressor.py`:
- 后处理压缩到目标 Token 预算
- 保护首尾回合，摘要中间部分
- 目标: 15K-29K tokens

## 六、数据流完整图示

```
用户交互
  │
  ▼
HermesCLI / GatewayRunner / ACP / TUI
  │
  ├─→ SessionDB (实时写入)
  │     └── sessions: 会话元数据
  │     └── messages: 消息历史
  │     └── messages_fts: 全文索引
  │
  ├─→ 记忆存储
  │     └── MEMORY.md / USER.md (Agent 自主更新)
  │
  ├─→ 日志
  │     └── agent.log / errors.log / gateway.log
  │
  ├─→ config.yaml (读取配置)
  │
  └─→ 技能目录 (按需读取)
        └── ~/.hermes/skills/
```

## 七、与 new-api 的数据层对比

| 方面 | new-api | Hermes Agent |
|------|---------|-------------|
| 数据库 | SQLite/MySQL/PostgreSQL 三选一 | 仅 SQLite |
| ORM | GORM v2 | 原生 sqlite3 |
| 缓存 | Redis + 内存 | 无外部缓存 |
| 全文搜索 | 无内置 | FTS5 (双分词器) |
| 会话存储 | 无会话概念（仅日志） | sessions + messages 表 |
| 配置 | Option 键值表 + 环境变量 | YAML 文件 + .env |
| 迁移 | GORM AutoMigrate | 手写模式对比 + 列添加 |
| 并发 | 连接池 + 事务 | WAL 模式 + 重试 |

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|Hermes Agent 项目架构总览]]
- [[02-agent-runtime|Agent 运行时引擎分析]]
- [[06-hermes-insights|Hermes 洞察与 Legion 参考]]

<!-- @end-section -->
