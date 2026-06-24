---
id: "analysis-deepseek-tui-storage-006"
title: "DeepSeek-TUI 会话管理与持久化"
aliases: ["deepseek-tui storage", "DeepSeek-TUI持久化"]
type: "analysis"
category: "design/analysis/deepseek-tui"
tags: ["deepseek-tui", "session", "sqlite", "storage", "config", "persistence"]
version: "1.0.0"
created: "2026-05-07"
updated: "2026-05-07"
author: "jxncyjq"
status: "review"
parent: "analysis-deepseek-tui-overview-001"
related_docs:
  - id: "analysis-deepseek-tui-overview-001"
    relation: "parent"
    path: "./01-overview.md"
  - id: "analysis-deepseek-tui-api-004"
    relation: "related_to"
    path: "./04-api-client.md"
  - id: "analysis-deepseek-tui-tools-005"
    relation: "related_to"
    path: "./05-tool-system.md"
---

# DeepSeek-TUI 会话管理与持久化

<!-- @section: overview -->

## 概述

DeepSeek-TUI 采用**分层持久化策略**：SQLite 存储结构化数据（线程、消息、检查点、任务），文件系统存储配置、记忆、会话索引和工作区快照。`crates/state` crate 负责所有数据库操作，`crates/config` crate 负责配置加载。

**存储位置总览**：`~/.deepseek/`（可通过 `DEEPSEEK_HOME` 覆盖）

<!-- @end-section -->

<!-- @section: sqlite-schema -->

## SQLite Schema

**数据库文件**：`~/.deepseek/sessions/state.db`（可通过 `DEEPSEEK_STATE_DB` 覆盖）

### 四张数据表

#### threads（会话线程）

```sql
CREATE TABLE threads (
    id TEXT PRIMARY KEY,                -- UUID
    rollout_path TEXT,                  -- 关联的文件路径
    preview TEXT NOT NULL,              -- 会话预览文本（首条消息摘要）
    ephemeral INTEGER NOT NULL,         -- 是否为临时会话（0/1）
    model_provider TEXT NOT NULL,       -- 提供商名称（deepseek/openai 等）
    created_at INTEGER NOT NULL,        -- Unix 时间戳（秒）
    updated_at INTEGER NOT NULL,
    status TEXT NOT NULL,               -- Running/Idle/Completed/Failed/Paused/Archived
    path TEXT,                          -- 关联文件路径
    cwd TEXT NOT NULL,                  -- 工作目录
    cli_version TEXT NOT NULL,          -- 创建时的 CLI 版本
    source TEXT NOT NULL,               -- Interactive/Resume/Fork/Api/Unknown
    title TEXT,                         -- 可选自定义标题
    sandbox_policy TEXT,                -- 安全策略快照
    approval_mode TEXT,                 -- 批准模式快照
    archived INTEGER DEFAULT 0,
    archived_at INTEGER,
    git_sha TEXT,                       -- 创建时的 Git commit SHA
    git_branch TEXT,
    git_origin_url TEXT,
    memory_mode TEXT                    -- 内存模式
);

CREATE INDEX idx_threads_updated_at ON threads(updated_at DESC);
CREATE INDEX idx_threads_archived_updated ON threads(archived, updated_at DESC);
```

#### messages（消息记录）

```sql
CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    thread_id TEXT NOT NULL,
    role TEXT NOT NULL,                -- "user" | "assistant"
    content TEXT NOT NULL,             -- 消息内容（JSON 序列化的 Vec<ContentBlock>）
    item_json TEXT,                    -- 扩展元数据（JSON）
    created_at INTEGER NOT NULL,
    FOREIGN KEY(thread_id) REFERENCES threads(id) ON DELETE CASCADE
);

CREATE INDEX idx_messages_thread_created_at ON messages(thread_id, created_at ASC);
```

**注意**：每个会话最多持久化 **500 条消息**（`MAX_PERSISTED_MESSAGES = 500`），超出部分通过上下文压缩处理。

#### checkpoints（会话快照）

```sql
CREATE TABLE checkpoints (
    thread_id TEXT NOT NULL,
    checkpoint_id TEXT NOT NULL,
    state_json TEXT NOT NULL,          -- 序列化的完整会话状态（JSON）
    created_at INTEGER NOT NULL,
    PRIMARY KEY(thread_id, checkpoint_id),
    FOREIGN KEY(thread_id) REFERENCES threads(id) ON DELETE CASCADE
);
```

**用途**：支持 `/session fork`（分叉会话）和会话状态回滚。

#### jobs（后台任务）

```sql
CREATE TABLE jobs (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    status TEXT NOT NULL,              -- Queued/Running/Completed/Failed/Cancelled
    progress INTEGER,                  -- 进度百分比（0-100）
    detail TEXT,                       -- 详细信息
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);
```

<!-- @end-section -->

<!-- @section: file-storage -->

## 文件存储结构

```
~/.deepseek/
├── config.toml                    # 主配置文件
├── sessions/
│   ├── state.db                   # SQLite 主数据库
│   ├── checkpoints/               # 检查点文件（JSON 快照）
│   └── offline_queue.json         # 离线消息队列
├── snapshots/
│   └── <project_hash>/            # 按项目哈希分区
│       └── <worktree_hash>/
│           └── .git/              # 工作区侧面 Git 副本
├── mcp.json                       # MCP 服务器配置
├── skills/                        # 用户技能/插件目录
├── memory.md                      # 用户持久记忆（Markdown）
├── notes.txt                      # 临时会话笔记
├── audit.log                      # 审计日志（追加写入）
└── tasks/                         # 后台任务工件目录
```

### 会话索引

**文件**：`session_index.jsonl`（每行一条 JSON 记录）

```jsonl
{"thread_id":"uuid-xxx","thread_name":"Fix login bug","updated_at":1746624000,"rollout_path":"/home/user/project"}
{"thread_id":"uuid-yyy","thread_name":"Add dark mode","updated_at":1746620000,"rollout_path":"/home/user/project"}
```

用于快速列出历史会话（无需全表扫描 SQLite）。

<!-- @end-section -->

<!-- @section: session-lifecycle -->

## 会话生命周期

### 创建与持久化

```rust
// 创建新会话
let session = Session::new(config, workspace);
state.create_thread(&session.metadata()).await?;

// 每轮对话后持久化
state.add_message(thread_id, &message).await?;
state.update_thread_status(thread_id, ThreadStatus::Idle).await?;

// 原子写入（防止数据损坏）
// 先写 temp 文件，再 rename 到目标路径
```

### 会话恢复

```rust
// 列出最近 50 个会话
let sessions = state.list_threads(ListThreadsQuery {
    limit: Some(50),
    archived: Some(false),
    ..Default::default()
}).await?;

// 恢复指定会话
let messages = state.get_messages(thread_id).await?;
let session = Session::restore(metadata, messages, config)?;
```

### 会话操作

| 操作 | 命令 | 说明 |
|------|------|------|
| 列出历史 | `/session list` | 显示最近 50 个会话 |
| 加载会话 | `/session load <id>` | 恢复历史会话 |
| 新建会话 | `/session new` | 清空状态，开始新会话 |
| 分叉会话 | `/session fork` | 基于当前状态创建分支 |
| 归档会话 | `/session archive` | 标记为已归档（不再显示） |

<!-- @end-section -->

<!-- @section: offline-queue -->

## 离线队列

**文件**：`~/.deepseek/sessions/offline_queue.json`

**用途**：网络故障时保存待发消息，防止用户输入丢失；重启后自动恢复消息队列。

```rust
pub struct OfflineQueueState {
    pub session_id: Option<String>,           // 关联的会话 ID
    pub messages: Vec<QueuedSessionMessage>,  // 待发送消息列表
    pub draft: Option<QueuedSessionMessage>,  // 当前编辑中的草稿
}

pub struct QueuedSessionMessage {
    pub id: String,          // 消息唯一 ID（用于幂等性验证）
    pub content: String,     // 消息内容
    pub queued_at: i64,      // 排队时间戳
}
```

**安全机制**：消息 ID 与会话 ID 绑定，防止旧会话的消息在新会话中意外重放。

<!-- @end-section -->

<!-- @section: cycle-checkpoint -->

## 循环检查点机制

**目的**：解决超长运行任务的上下文窗口溢出问题，类似于"分章节"写作。

### 检查点触发

```rust
pub struct CycleConfig {
    pub turns_per_cycle: u32,  // 每循环的回合数（默认 25）
}

pub fn should_advance_cycle(turn_count: u32, config: &CycleConfig) -> bool {
    turn_count >= config.turns_per_cycle
}
```

### 循环状态

```rust
pub struct Session {
    pub cycle_count: u32,                      // 已完成的循环数
    pub current_cycle_started: DateTime<Utc>,  // 当前循环开始时间
    pub cycle_briefings: Vec<CycleBriefing>,   // 历史循环摘要
}

pub struct CycleBriefing {
    pub cycle_index: u32,
    pub summary: String,         // LLM 生成的本循环摘要
    pub turn_count: u32,         // 本循环执行的回合数
    pub completed_at: DateTime<Utc>,
}
```

### 检查点流程

```
turn_count >= 25
  ↓
archive_cycle()
  ├── 调用 deepseek-v4-flash 生成本循环摘要
  ├── 摘要存入 cycle_briefings
  ├── 重置消息缓冲区（保留系统提示和摘要）
  └── cycle_count += 1
  ↓
继续下一循环（从干净的上下文开始）
```

<!-- @end-section -->

<!-- @section: memory-system -->

## 用户记忆系统

**文件**：`~/.deepseek/memory.md`

**启用方式**：
```toml
# ~/.deepseek/config.toml
[memory]
enabled = true
```
或 `DEEPSEEK_MEMORY=on`

### 记忆格式

Markdown 格式，时间戳列表项：

```markdown
# User Memory

- [2026-05-01] 我的主要项目是 Legion，Go 微服务框架
- [2026-05-02] 偏好使用 PostgreSQL 而不是 MySQL
- [2026-05-03] 不喜欢过度注释，代码要简洁
- [2026-05-07] 团队采用 Conventional Commits 规范
```

### 注入位置

记忆内容注入系统提示词中的 `<user_memory>` 标签：

```xml
<user_memory>
- [2026-05-01] 我的主要项目是 Legion，Go 微服务框架
- [2026-05-07] 团队采用 Conventional Commits 规范
</user_memory>
```

### 记忆管理

| 操作 | 方式 |
|------|------|
| 添加记忆 | `/memory add <text>` 或对话中说 "# note ..." |
| 查看记忆 | `/memory` 命令 |
| 编辑记忆 | `/memory edit`（打开 `$EDITOR`） |
| `remember` 工具 | LLM 主动调用，将重要信息写入记忆 |

<!-- @end-section -->

<!-- @section: config-reference -->

## 完整配置参考

### 配置文件位置

1. `~/.deepseek/config.toml`（用户级，最常用）
2. `.deepseek/config.toml`（项目级，覆盖用户级）
3. 环境变量 `DEEPSEEK_*`（最高优先级，除 CLI 外）

### 完整配置项

```toml
# === 提供商与认证 ===
provider = "deepseek"             # deepseek|nvidia-nim|openrouter|novita|fireworks|sglang
api_key = "sk-xxxxxxxx"
base_url = "https://api.deepseek.com"
default_text_model = "deepseek-v4-pro"

# === 模型参数 ===
reasoning_effort = "high"         # off|low|medium|high|max

# === 路径 ===
skills_dir = "~/.deepseek/skills"
mcp_config_path = "~/.deepseek/mcp.json"
notes_path = "~/.deepseek/notes.txt"
memory_path = "~/.deepseek/memory.md"
instructions = ["./AGENTS.md", "~/.deepseek/global.md"]

# === 安全策略 ===
allow_shell = true
approval_policy = "on-request"    # on-request|untrusted|never
sandbox_mode = "workspace-write"  # read-only|workspace-write|danger-full-access|external-sandbox
auto_allow = ["git status", "cargo check"]
max_subagents = 10                # 并发子代理上限（1-20）

# === 用户记忆 ===
[memory]
enabled = true

# === TUI 设置 ===
[tui]
alternate_screen = "auto"         # auto|always|never
mouse_capture = true              # true=仅文本复制，false=原始终端选择
osc8_links = true                 # 终端 URL 点击支持
language = "zh-Hans"              # 界面语言覆盖

# === 功能开关 ===
[features]
shell_tool = true
subagents = true
web_search = true
apply_patch = true
mcp = true

# === 网络策略 ===
[network]
default = "prompt"                # allow|deny|prompt
allow = ["api.deepseek.com"]
deny = []
audit = true

# === 重试策略 ===
[retry]
enabled = true
max_retries = 3
initial_delay = 1.0               # 秒
max_delay = 60.0

# === 上下文压缩 ===
[context]
enabled = false
verbatim_window_turns = 16        # 保留最近 N 轮完整内容
l1_threshold = 192000
l2_threshold = 384000
l3_threshold = 576000
cycle_threshold = 768000
seam_model = "deepseek-v4-flash"  # 压缩用的轻量模型

# === 容量控制 ===
[capacity]
enabled = false
low_risk_max = 0.50               # 低风险操作的最大容量比
medium_risk_max = 0.62

# === 多提供商配置 ===
[providers.deepseek]
api_key = "sk-xxxxxxxx"
base_url = "https://api.deepseek.com"
model = "deepseek-v4-pro"

[providers.openrouter]
api_key = "sk-or-xxxxxxxx"
base_url = "https://openrouter.ai/api/v1"
model = "deepseek/deepseek-v4-pro"
```

### 关键环境变量

```bash
DEEPSEEK_API_KEY           # API 密钥
DEEPSEEK_BASE_URL          # API 端点
DEEPSEEK_PROVIDER          # 提供商名称
DEEPSEEK_MODEL             # 模型 ID
DEEPSEEK_ALLOW_SHELL       # 允许 Shell（true/false）
DEEPSEEK_APPROVAL_POLICY   # 批准策略
DEEPSEEK_SANDBOX_MODE      # 沙盒模式
DEEPSEEK_MEMORY            # 记忆系统（on/off）
DEEPSEEK_HOME              # 配置根目录（覆盖 ~/.deepseek）
DEEPSEEK_STATE_DB          # SQLite 数据库路径
DEEPSEEK_FORCE_HTTP1       # 强制 HTTP/1.1
RUST_LOG                   # 日志级别（info/debug/trace）
```

<!-- @end-section -->

<!-- @section: related -->

## 相关文档

- [[01-overview|项目总览]]
- [[04-api-client|API 客户端与流式处理]]
- [[05-tool-system|工具系统与 MCP]]
- [[07-insights|设计洞察与 Legion 参考]]

<!-- @end-section -->
