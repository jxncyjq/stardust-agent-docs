---
id: "analysis-deepseek-tui-tools-005"
title: "DeepSeek-TUI 工具系统与 MCP"
aliases: ["deepseek-tui tools", "DeepSeek-TUI工具系统"]
type: "analysis"
category: "design/analysis/deepseek-tui"
tags: ["deepseek-tui", "tools", "mcp", "sandbox", "approval", "subagent"]
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
  - id: "analysis-deepseek-tui-crates-002"
    relation: "related_to"
    path: "./02-crate-analysis.md"
  - id: "analysis-deepseek-tui-api-004"
    relation: "related_to"
    path: "./04-api-client.md"
---

<!-- @section: overview -->

# DeepSeek-TUI 工具系统与 MCP

## 概述

DeepSeek-TUI 的工具系统允许 LLM 执行真实的操作：运行 Shell 命令、读写文件、搜索网页、调用 GitHub API、管理任务列表，以及启动子代理进行并行处理。工具系统由 **execpolicy crate**（策略引擎）、**tools crate**（类型定义）和 **tui crate 中的路由层**（`tool_routing.rs`）三部分构成。

**9 种内置工具**：shell, file (read/write/glob), git, web_search, github, tasks, subagent, rlm, mcp（外部）

<!-- @end-section -->

<!-- @section: data-structures -->

## 工具数据结构

```rust
// 工具定义（发送给 LLM 的工具规格）
pub struct Tool {
    pub name: String,                     // 工具名称
    pub description: String,              // 工具描述（LLM 根据此选择工具）
    pub input_schema: Value,              // JSON Schema（定义参数结构）
    pub tool_type: Option<String>,        // "function"
    pub strict: Option<bool>,             // 严格模式（参数校验）
    pub cache_control: Option<CacheControl>, // 前缀缓存控制
}

// 工具调用（LLM 发出的工具调用请求）
pub struct ToolInvocation {
    pub id: String,         // 唯一 ID（用于关联结果）
    pub name: String,       // 工具名称
    pub input: Value,       // 参数（JSON）
}

// 工具批准请求
pub struct ApprovalRequest {
    pub tool_name: String,
    pub input: Value,
    pub reason: Option<String>,     // 为何需要批准
    pub risk_level: RiskLevel,      // Low | Medium | High
}

// 批准决策
pub enum ReviewDecision {
    Allow,          // 本次允许
    AllowOnce,      // 仅此一次
    AllowAlways,    // 永远允许（加入 auto_allow）
    Deny,           // 拒绝
}
```

<!-- @end-section -->

<!-- @section: tool-routing -->

## 工具路由系统

**文件**：`crates/tui/src/tool_routing.rs`

### 路由流程

```
LLM 发出 ToolUse ContentBlock
  ↓
[工具名称解码] from_api_tool_name()
  ↓
[工具查找] tool_registry.get(name)
  ↓
[参数校验] JSON Schema 验证
  ↓
[策略检查] execpolicy::should_approve(tool, input, sandbox)
  │
  ├── auto_allow 列表匹配 → 直接执行
  ├── read-only 操作 → 直接执行
  ├── 需要批准 → 弹出 approval.rs 对话框 → 等待用户决策
  └── 被拒绝 → 返回错误 ToolResult
  ↓
[并发执行] 最多 4 个工具并行
  │
  ├── shell → shell_job_routing.rs → portable-pty
  ├── file → 文件系统操作
  ├── git → git CLI 封装
  ├── web_search → HTTP 搜索 API
  ├── github → GitHub REST API
  ├── tasks → 本地 TODO 管理
  ├── subagent → subagent_routing.rs
  ├── rlm → Python REPL 子进程
  └── mcp_* → mcp_routing.rs → JSON-RPC
  ↓
[结果流式返回]
  ├── streaming/chunking.rs（块级分割）
  ├── streaming/line_buffer.rs（行级缓冲）
  └── streaming/commit_tick.rs（时序控制）
  ↓
[后置钩子]
  ├── LSP 诊断注入（文件修改后）
  ├── 文件频率更新（file_frecency.rs）
  └── hooks.rs post_tool_call 回调
```

<!-- @end-section -->

<!-- @section: builtin-tools -->

## 内置工具清单

### shell

**职责**：在工作区执行 Shell 命令，使用伪终端（PTY）以获得完整的 TTY 行为。

```json
{
  "name": "shell",
  "description": "Execute a shell command in the workspace",
  "input_schema": {
    "type": "object",
    "properties": {
      "command": { "type": "string" },
      "timeout_ms": { "type": "integer", "default": 60000 },
      "cwd": { "type": "string" }
    },
    "required": ["command"]
  }
}
```

**安全控制**：
- `allow_shell = false`（配置）→ 工具被禁用
- `sandbox_mode = read-only` → shell 工具不可用
- `approval_policy = on-request` → 高风险命令需要批准

---

### file 工具族

| 工具名 | 功能 |
|--------|------|
| `read_file` | 读取文件内容（支持行号范围） |
| `write_file` | 写入完整文件内容 |
| `patch_file` | 应用 unified diff 补丁 |
| `delete_file` | 删除文件 |
| `create_directory` | 创建目录 |
| `glob` | 文件名模式匹配搜索 |
| `grep` | 正则内容搜索（使用 `ignore` 库，尊重 .gitignore） |

---

### git

**职责**：Git 操作封装（通过 CLI）。

常用操作：`git_status`, `git_diff`, `git_log`, `git_commit`, `git_checkout`, `git_branch`, `git_show`

---

### web_search

**职责**：搜索网络内容，获取最新信息。

实现方式：调用配置的搜索 API（支持 Brave Search、SerpAPI 等），解析结果并返回摘要。

---

### github

**职责**：GitHub API 操作（需要 `GITHUB_TOKEN`）。

操作：查看 issue、创建 PR、搜索代码、获取文件内容等。

---

### tasks

**职责**：本地 TODO 列表管理，在 `sidebar.rs` 中显示进度。

操作：`task_add`, `task_complete`, `task_list`, `task_delete`

---

### subagent

**职责**：启动子代理，在独立上下文中并行处理任务。

```rust
// 并发控制
pub struct SubagentConfig {
    pub max_concurrent: u32,  // 1-20，默认 10
}

// 子代理状态（显示在 sidebar 的 Agents 面板）
pub enum SubagentStatus {
    Queued,
    Running { progress: Option<String> },
    Completed { result: String },
    Failed { error: String },
}
```

**使用场景**：并行处理多个独立文件、批量代码审查、多文档生成。

---

### rlm（递归语言模型）

**职责**：启动轻量级 `deepseek-v4-flash` 子运行时，执行 Python 代码，用于数据分析、计算任务。

```
主代理
  └── rlm 工具调用
        └── deepseek-v4-flash 子实例（1-16 个并行）
              └── Python REPL 沙盒
```

<!-- @end-section -->

<!-- @section: approval-sandbox -->

## 批准策略与沙盒

### 批准策略

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| `on-request` | 高风险工具操作需要用户批准 | 默认模式，开发环境 |
| `untrusted` | 仅第三方工具（MCP）需要批准 | 信任本地工具 |
| `never` | 从不请求批准（完全 YOLO）| 自动化 CI/CD |

**风险分级**：
- **低风险**：read_file, glob, grep, git_status（自动允许）
- **中风险**：write_file, git_commit, web_search（`on-request` 下询问）
- **高风险**：shell（任意命令）、delete_file（`on-request` 下必须批准）

### 沙盒模式

| 模式 | 文件读 | 文件写 | Shell | 说明 |
|------|--------|--------|-------|------|
| `read-only` | ✅ | ❌ | ❌ | 仅查看，不能修改 |
| `workspace-write` | ✅ | 仅工作区 | ✅（需批准） | **默认模式** |
| `danger-full-access` | ✅ | ✅（全系统） | ✅ | 完全访问 |
| `external-sandbox` | ✅ | 受限 | 沙盒内 | macOS Seatbelt |

### auto_allow 机制

配置可信命令前缀，自动跳过批准弹窗：

```toml
# ~/.deepseek/config.toml
auto_allow = [
    "git status",
    "git diff",
    "cargo check",
    "cargo test",
    "ls",
    "cat",
]
```

### 批准 UI（approval.rs - 59K）

批准对话框显示：
- 工具名称和风险级别
- 完整参数（语法高亮）
- 风险说明
- 快捷键操作（`y` 允许，`n` 拒绝，`a` 永远允许）

<!-- @end-section -->

<!-- @section: mcp -->

## MCP 集成

### Model Context Protocol (MCP)

**配置文件**：`~/.deepseek/mcp.json`

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"],
      "env": {}
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": { "DATABASE_URL": "postgresql://..." }
    }
  }
}
```

**工具命名规则**：`mcp_<server_name>_<tool_name>`
（如：`mcp_filesystem_read_file`, `mcp_postgres_query`）

**传输方式**：
- **stdio**：启动子进程，通过 stdin/stdout 通信（最常用）
- **HTTP**：JSON-RPC over HTTP（适合远程服务器）

**生命周期管理（mcp crate）**：
- 启动时初始化所有配置的 MCP 服务器
- 健康检查（心跳）
- 错误时自动重启
- 应用退出时清理子进程

<!-- @end-section -->

<!-- @section: lsp-snapshot -->

## LSP 诊断集成

**目录**：`crates/tui/src/lsp/`

每次文件修改（`write_file`, `patch_file`, `apply_patch`）后，自动触发 LSP 诊断，将编译错误/警告注入下一轮对话。

### 支持的语言服务器

| 语言 | 服务器 | 默认命令 |
|------|--------|---------|
| Rust | rust-analyzer | `rust-analyzer` |
| Python | pyright | `pyright-langserver --stdio` |
| Go | gopls | `gopls` |
| TypeScript/JS | tsserver | `typescript-language-server --stdio` |
| C/C++ | clangd | `clangd` |
| Java | jdtls | `jdtls` |

### 工作区快照

**目的**：支持 `/restore` 命令和 `revert_turn` 工具，可将工作区回滚到任意历史状态。

**存储路径**：`~/.deepseek/snapshots/<project_hash>/<worktree_hash>/.git`

**快照标签**：
- `pre-turn:<seq>` — 工具执行前的状态
- `post-turn:<seq>` — 工具执行后的状态

**特点**：使用独立的 `--git-dir` + `--work-tree`，不影响项目自身的 `.git` 目录。

<!-- @end-section -->

<!-- @section: related -->

## 相关文档

- [[01-overview|项目总览]]
- [[02-crate-analysis|Crate 职责分析]]
- [[04-api-client|API 客户端与流式处理]]
- [[06-session-storage|会话管理与持久化]]

<!-- @end-section -->
