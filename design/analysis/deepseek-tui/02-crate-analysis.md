---
id: "analysis-deepseek-tui-crates-002"
title: "DeepSeek-TUI Crate 职责分析"
aliases: ["deepseek-tui crates", "DeepSeek-TUI模块分析"]
type: "analysis"
category: "design/analysis/deepseek-tui"
tags: ["deepseek-tui", "rust", "crates", "architecture", "modules"]
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
  - id: "analysis-deepseek-tui-tui-003"
    relation: "related_to"
    path: "./03-tui-components.md"
  - id: "analysis-deepseek-tui-api-004"
    relation: "related_to"
    path: "./04-api-client.md"
---

<!-- @section: overview -->

# DeepSeek-TUI Crate 职责分析

## 工作区概览

DeepSeek-TUI 采用 Cargo 工作区组织，共 14 个 crate，每个 crate 单一职责。工作区版本统一为 **v0.8.11**，使用 Rust edition 2024。

```
crates/
├── agent/      # 供应商注册表与模型路由
├── app-server/ # HTTP/SSE 传输层
├── cli/        # CLI 入口 dispatcher
├── config/     # 配置加载与验证
├── core/       # 核心代理循环与引擎
├── execpolicy/ # 批准/沙盒策略引擎
├── hooks/      # 生命周期钩子
├── mcp/        # MCP 客户端
├── protocol/   # 数据结构与协议框架
├── secrets/    # OS 密钥管理
├── state/      # SQLite 持久化层
├── tools/      # 工具原始类型
├── tui/        # 主 TUI 应用（最大 crate）
└── tui-core/   # 事件驱动状态机框架
```

<!-- @end-section -->

<!-- @section: layer0 -->

## Layer 0：基础层

### protocol

**职责**：定义所有数据结构和协议框架，是整个系统的数据契约层。

**关键类型**：
```rust
// 消息数据结构
pub struct Message {
    pub role: String,                // "user" | "assistant"
    pub content: Vec<ContentBlock>, // 内容块数组
}

pub enum ContentBlock {
    Text { text: String, cache_control: Option<CacheControl> },
    Thinking { thinking: String },
    ToolUse { id: String, name: String, input: Value, caller: Option<ToolCaller> },
    ToolResult { tool_use_id: String, content: String, is_error: Option<bool> },
    ServerToolUse { id: String, name: String, input: Value },
}

// 线程/会话元数据
pub struct ThreadMetadata {
    pub id: String,
    pub preview: String,
    pub ephemeral: bool,
    pub model_provider: String,
    pub created_at: i64,
    pub updated_at: i64,
    pub status: ThreadStatus,    // Running/Idle/Completed/Failed/Paused/Archived
    pub cwd: PathBuf,
    pub cli_version: String,
    pub source: SessionSource,   // Interactive/Resume/Fork/Api
    pub git_sha: Option<String>,
    pub git_branch: Option<String>,
}

pub enum SystemPrompt {
    Text(String),
    Blocks(Vec<SystemBlock>),
}
```

**对外接口**：所有 crate 均依赖 protocol 中的数据结构进行通信。

---

### config

**职责**：TOML 配置文件加载、环境变量覆盖、配置优先级管理、JSON Schema 生成。

**配置加载优先级（从低到高）**：
1. 内置默认值
2. 用户级 `~/.deepseek/config.toml`
3. 项目级 `.deepseek/config.toml`
4. 环境变量（`DEEPSEEK_*` 前缀）
5. CLI 参数

**核心配置结构**：
```rust
pub struct Config {
    pub provider: ProviderKind,           // 活跃提供商
    pub api_key: String,
    pub base_url: String,
    pub default_text_model: String,
    pub reasoning_effort: ReasoningEffort, // off|low|medium|high|max
    pub allow_shell: bool,
    pub approval_policy: ApprovalPolicy,   // on-request|untrusted|never
    pub sandbox_mode: SandboxMode,         // read-only|workspace-write|danger-full-access
    pub auto_allow: Vec<String>,
    pub skills_dir: PathBuf,
    pub mcp_config_path: PathBuf,
    pub memory_path: PathBuf,
    pub instructions: Vec<PathBuf>,
    pub providers: ProvidersToml,          // 多提供商配置
    pub tui: Option<TuiConfig>,
}
```

---

### state

**职责**：SQLite 持久化层，管理会话线程、消息记录、检查点快照、后台任务。

**数据库位置**：`~/.deepseek/sessions/state.db`（可通过 `DEEPSEEK_STATE_DB` 配置）

**4 张数据表**：

| 表名 | 主要字段 | 说明 |
|------|---------|------|
| threads | id, preview, status, cwd, model_provider, created_at | 会话线程元数据 |
| messages | id(auto), thread_id, role, content, item_json, created_at | 消息记录 |
| checkpoints | thread_id, checkpoint_id, state_json, created_at | 会话快照 |
| jobs | id, name, status, progress, created_at | 后台任务 |

```sql
-- 关键索引
CREATE INDEX idx_threads_updated_at ON threads(updated_at DESC);
CREATE INDEX idx_threads_archived_updated ON threads(archived, updated_at DESC);
CREATE INDEX idx_messages_thread_created_at ON messages(thread_id, created_at ASC);
```

---

### tui-core

**职责**：事件驱动的 TUI 状态机框架，提供基础 Widget 抽象、布局约束、消息总线。

**核心抽象**：
```rust
pub trait Component {
    type Msg;
    fn update(&mut self, msg: Self::Msg) -> Option<Self::Msg>;
    fn render(&self, frame: &mut Frame, area: Rect);
}
```

<!-- @end-section -->

<!-- @section: layer1 -->

## Layer 1：扩展层

### tools

**职责**：工具调用的原始类型定义、工具规格（JSON Schema）、调用记录。

```rust
pub struct Tool {
    pub name: String,
    pub description: String,
    pub input_schema: Value,         // JSON Schema
    pub tool_type: Option<String>,
    pub strict: Option<bool>,
    pub cache_control: Option<CacheControl>,
}

pub struct ToolInvocation {
    pub id: String,
    pub name: String,
    pub input: Value,
}
```

---

### execpolicy

**职责**：批准/沙盒策略引擎，决定工具是否需要用户批准，控制文件系统访问权限。

**批准策略**：
```rust
pub enum ApprovalPolicy {
    OnRequest,   // 高风险工具需要批准
    Untrusted,   // 第三方工具（MCP）需要批准
    Never,       // 从不批准（完全手动）
}

pub enum SandboxMode {
    ReadOnly,             // 文件系统只读
    WorkspaceWrite,       // 仅工作区可写（默认）
    DangerFullAccess,     // 完全访问
    ExternalSandbox,      // 外部沙盒（macOS Seatbelt）
}
```

---

### hooks

**职责**：生命周期钩子系统，在工具执行前后、会话开始/结束等关键节点触发回调。

**钩子类型**：
- `pre_tool_call` — 工具调用前
- `post_tool_call` — 工具调用后（含 LSP 诊断注入）
- `session_start` / `session_end`
- `file_modified` — 文件修改后触发文件频率更新

---

### mcp

**职责**：Model Context Protocol (MCP) 客户端，管理外部工具服务器生命周期，代理工具调用。

**支持传输方式**：
- **stdio** — 子进程 + 标准输入/输出
- **HTTP** — JSON-RPC over HTTP

**工具命名规则**：`mcp_<server>_<tool_name>`

**配置文件**：`~/.deepseek/mcp.json`

<!-- @end-section -->

<!-- @section: layer2-3 -->

## Layer 2：代理层 / Layer 3：引擎层

### agent

**职责**：LLM 提供商注册表，管理 7 个 ApiProvider 的 endpoint/model 映射，处理提供商特定行为。

```rust
pub enum ApiProvider {
    Deepseek,       // api.deepseek.com（默认）
    DeepseekCN,     // 中国区 API
    Openrouter,     // openrouter.ai/api/v1
    Novita,         // api.novita.ai/v1
    Fireworks,      // api.fireworks.ai/inference/v1
    NvidiaNim,      // integrate.api.nvidia.com/v1
    Sglang,         // localhost:30000/v1（自托管）
}

// 上下文窗口检测
pub fn context_window_for_model(model: &str) -> Option<u32> {
    match model {
        m if m.contains("v4") => Some(1_000_000),  // 1M tokens
        m if m.contains("v3") => Some(128_000),    // 128K tokens
        _ => None,
    }
}
```

---

### core

**职责**：核心代理循环，是整个系统最复杂的 crate（`engine.rs` 约 74K 行代码）。

**主要子模块**：

| 子模块 | 职责 |
|--------|------|
| `engine.rs` | 代理主循环，Turn 编排，工具调用路由 |
| `session.rs` | 会话状态管理，消息历史维护 |
| `turn.rs` | 单次对话回合的完整生命周期 |
| `capacity.rs` | 容量控制器，防止 token 溢出 |
| `coherence.rs` | 一致性状态跟踪 |

**会话结构**：
```rust
pub struct Session {
    pub model: String,
    pub reasoning_effort: Option<String>,
    pub messages: Vec<Message>,
    pub system_prompt: Option<SystemPrompt>,
    pub total_usage: SessionUsage,
    pub workspace: PathBuf,
    pub id: String,
    pub cycle_count: u32,                    // 已跨越的循环数
    pub cycle_briefings: Vec<CycleBriefing>, // 历史摘要
    pub working_set: WorkingSet,             // 活跃编辑文件集
    pub project_context: ProjectContext,     // AGENTS.md 内容
}
```

**Turn 状态机**：
```
NoStream
  ↓
Streaming（SSE 接收中）
  ├── ContentBlockStart → ContentBlockDelta → ContentBlockStop
  └── 存储 TextBlock, ThinkingBlock, ToolUseBlock[]
  ↓
ToolExecution（并发执行最多 4 个工具）
  └── 收集结果 → ToolResult
  ↓
ResponseProcessed（检查 stop_reason）
  └── end_turn / tool_calls / max_tokens
  ↓
TurnComplete → 保存会话
```

<!-- @end-section -->

<!-- @section: layer4-5 -->

## Layer 4：应用层 / Layer 5：顶层

### app-server

**职责**：HTTP/SSE 传输层，提供 REST API，允许外部工具通过 HTTP 与 DeepSeek-TUI 交互。

**技术栈**：axum 0.8.5 + tower-http（CORS）

**主要端点**：
- `POST /v1/messages` — 创建消息请求
- `GET /v1/threads` — 列出会话线程
- `GET /v1/threads/:id` — 获取会话详情
- `SSE /v1/events` — 实时事件流

---

### tui

**职责**：主 TUI 应用，包含 52 个模块，230+ Rust 源文件。这是整个项目最大的 crate，集成了 Ratatui 渲染、用户交互、工具路由等所有 UI 相关逻辑。详见 [[03-tui-components|TUI 组件系统]]。

**二进制名**：`deepseek-tui`

---

### cli

**职责**：顶层 CLI dispatcher，根据参数决定启动 TUI 模式还是 app-server 模式。

**二进制名**：`deepseek`

**入口**：
```rust
// crates/cli/src/main.rs
fn main() -> std::process::ExitCode {
    deepseek_tui_cli::run_cli()
}
```

**路由逻辑**：
- `deepseek` — 进入 TUI 交互模式
- `deepseek serve` — 启动 app-server
- `deepseek --model <id> "<prompt>"` — 单次模式
- `deepseek --profile <name>` — 加载指定配置 profile

---

### secrets

**职责**：OS keyring 密钥管理，提供跨平台的 API 密钥安全存储，文件系统回退。

**平台支持**：
- macOS：Keychain（apple-native feature）
- Windows：Windows Credential Manager（windows-native feature）
- Linux：libsecret / keyutils（linux-native-sync-persistent feature）

<!-- @end-section -->

<!-- @section: related -->

## 相关文档

- [[01-overview|项目总览]]
- [[03-tui-components|TUI 组件系统]]
- [[04-api-client|API 客户端与流式处理]]

<!-- @end-section -->
