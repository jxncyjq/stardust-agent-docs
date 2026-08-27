---
id: "reference-legion-agent-config-context-001"
title: "Legion Agent 配置与上下文文件"
aliases: ["agent.json", "上下文文件", "AGENTS SOUL TOOLS USER MEMORY"]
type: "reference"
category: "agents/reference"
tags: ["agent", "configuration", "context-files", "maas", "persona"]
version: "2.0.0"
created: "2026-05-19"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "agent-configuration-001"
    relation: "related_to"
    path: "../legion-agent/configuration.md"
  - id: "reference-maas-model-profiles-001"
    relation: "related_to"
    path: "../legion-agent/model-profiles.md"
  - id: "reference-legion-agent-auth-001"
    relation: "related_to"
    path: "./reference-legion-agent-auth-001.md"
  - id: "agent-persona-files-001"
    relation: "related_to"
    path: "../legion-agent/persona-files.md"
---

# Legion Agent 配置与上下文文件

默认配置文件是 `agent.json`（`--config` 指定路径）。完整字段样例见仓库里的 `configs/agent.complete.example.json` 与 `configs/agent.full.example.json`。

<!-- @section: blocks -->
## 配置块总览

| 配置块 | 作用 |
|--------|------|
| `maas` | 模型服务地址、API key、profile 表、默认 profile |
| `agents` | 子 Agent 注册表（名字 → 配置文件路径） |
| `storage` | 存储驱动与路径（`sqlite` / `memory`） |
| `server` | HTTP 监听、token、身份要求、loopback 加固、文件外链基址 |
| `service` | 后台调度间隔 |
| `runtime` | 工具轮数、并发、lazy tools、压缩阈值、审批超时、禁用工具、调试探针 |
| `session` | 会话连续性与上下文缓存 |
| `tui` | TUI 显示与配色 |
| `skills` | 技能 registry 与安装目录 |
| `context_files` | 常驻上下文文件（persona 系列） |
| `workspace` | 会话状态、docs、memory 的根目录 |
| `tasks` | 多 Agent 共享任务账本路径与膨胀控制 |
| `web` | `fetch_url` / `web_extract` / `web_search` 工具 |
| `browser` | 内置浏览器运行时 |
| `evolution` | 演化/自我改进相关开关 |
| `plugins` | WASM 插件部署清单、缓存、资源上限、签名要求 |

<!-- @end-section -->

<!-- @section: minimal -->
## 最小配置骨架

```json
{
  "maas": {
    "default_profile": "dev",
    "profiles": {
      "dev": { "base_url": "https://api.example.com", "model": "deepseek-chat", "api_key": "change-me" }
    }
  },
  "storage": { "driver": "sqlite", "path": "agent.db" },
  "server": { "listen_addr": ":8080", "admin_token": "change-me", "public_health_enabled": true },
  "runtime": { "max_tool_rounds": 4, "lazy_tools": true, "max_concurrent_tasks": 4 },
  "session": {
    "enabled": true,
    "default_recent_turns": 6,
    "max_turn_chars": 6000,
    "restore_latest_on_tui_start": true,
    "cache_enabled": true,
    "cache_max_entries": 128
  }
}
```

`storage.driver=sqlite` 是推荐值：只有 SQLite 模式才持久化 task、event、audit、session、message。`memory` 驱动重启即失忆，只适合 demo。

<!-- @end-section -->

<!-- @section: maas -->
## MaaS 与模型 profile

```json
{
  "maas": {
    "default_profile": "dev",
    "profiles": {
      "dev": {
        "base_url": "https://api.example.com",
        "model": "deepseek-chat",
        "api_key": "change-me",
        "context_length": 65536,
        "prompt_cache": false
      }
    }
  }
}
```

| 字段 | 说明 |
|------|------|
| `model` | 非空 → 调 `{base_url}/chat/completions`（OpenAI 兼容）；为空 → 走 Legion 内部 MaaS 协议 `{base_url}/v1/inference/generate` |
| `context_length` | 模型上下文窗口，供 GUI 展示；provider 的 `/models` 不返回这个值，必须手填，0 表示未设 |
| `prompt_cache` | 是否给稳定前缀打 `cache_control` 断点。**DeepSeek 实测无效**（它按最长公共 token 前缀自动缓存，且不认部分头部命中），在 DeepSeek 上真正省钱的是保持前缀稳定而不是打标记 |

profile 选择优先级：

| 来源 | 说明 |
|------|------|
| `--maas-profile` | 命令行显式指定，最高 |
| 子 Agent 的 `maas_profile` | `@researcher`、workflow `agent_id` 命中子 Agent 时 |
| `maas.default_profile` | 默认 |
| `--maas-url` | 临时覆盖 base URL，并绕过 profile client 构建 |

API key 不要写进可提交的配置文件；用环境变量或未提交的本地文件。

<!-- @end-section -->

<!-- @section: server-runtime -->
## server 与 runtime

```json
{
  "server": {
    "listen_addr": ":8080",
    "admin_token": "change-me",
    "public_health_enabled": true,
    "require_identity": false,
    "request_id_header": "X-Request-ID",
    "loopback_hardening": false,
    "handshake_file": "",
    "file_base_url": ""
  },
  "runtime": {
    "demo_response": "task completed",
    "max_tool_rounds": 4,
    "compact_token_threshold": 0,
    "lazy_tools": true,
    "max_concurrent_tasks": 4,
    "approval_timeout_seconds": 300,
    "disabled_tools": [],
    "debug": false
  }
}
```

| 字段 | 默认 | 说明 |
|------|------|------|
| `server.listen_addr` | `:8080` | 为空且未传 `--addr` 时绑 `127.0.0.1:0` 并自动 loopback 加固 |
| `server.require_identity` | false | true 时 `X-Role` / `X-Company-ID` 必填，详见 [[reference-legion-agent-auth-001\|鉴权参考]] |
| `server.loopback_hardening` | false | 一次性 token + 握手文件 + Origin 守卫 |
| `server.file_base_url` | 空 | 生成文件外链基址；空时返回相对 `/v1/files?...` |
| `runtime.max_tool_rounds` | 4 | 最大连续工具轮数 |
| `runtime.lazy_tools` | true | 元工具按需加载协议；false 回退全量 schema |
| `runtime.max_concurrent_tasks` | 4 | 并发任务上限 |
| `runtime.compact_token_threshold` | 0（关闭） | 超阈值压缩会话，单任务最多 3 次 |
| `runtime.approval_timeout_seconds` | 300 | Manual 审批超时自动拒绝 |
| `runtime.disabled_tools` | 空 | per-agent 工具禁用清单，名字必须是 gateable 工具 |
| `runtime.debug` | false | 推理前打印每条消息的角色/字符数/工具调用数，定位 prompt 膨胀 |

<!-- @end-section -->

<!-- @section: tools-config -->
## web、browser 与 plugins

```json
{
  "web": {
    "enabled": true,
    "allow_private_hosts": false,
    "timeout_seconds": 20,
    "max_response_kb": 512,
    "allowlist": ["example.com"],
    "searxng_url": "http://127.0.0.1:9888",
    "search_engine": "baidu",
    "search_default_limit": 5,
    "search_timeout_seconds": 15
  },
  "browser": {
    "enabled": false,
    "headless": true,
    "bin_path": "",
    "session_ttl_seconds": 600,
    "reap_interval_seconds": 60,
    "max_elements": 100,
    "snapshot_rune_threshold": 20000,
    "snapshot_ttl_hours": 24,
    "snapshot_archive_dir": ""
  },
  "plugins": { "manifest": "" }
}
```

- `web.searxng_url` 为空时 `web_search` **根本不注册**，模型看不到这个工具。
- `browser.enabled=false` 是默认；开启需要环境里有可用 Chromium，`bin_path` 可指向系统 Chrome/Edge 绕过自动下载。
- `browser.snapshot_rune_threshold` 为 0 时关闭观测降级；非 0 时超阈值的页面文本会落盘（去重）并做任务导向抽取，模型再用 `read_file` 翻页。
- `plugins.manifest` 为空 = 这套部署不跑插件（契约声明的可选项）；配了路径但读不了/解析不了，`serve` 直接启动失败。相对路径按**进程工作目录**解析，不是配置文件所在目录。

<!-- @end-section -->

<!-- @section: session -->
## session 与上下文缓存

```json
{
  "session": {
    "enabled": true,
    "default_recent_turns": 6,
    "max_turn_chars": 6000,
    "restore_latest_on_tui_start": true,
    "cache_enabled": true,
    "cache_max_entries": 128
  }
}
```

`cache_enabled` 只影响最近 turns 的**进程内**缓存，不影响 `agent_sessions` / `conversation_turns` 的持久化。调试上下文刷新时可临时设 false。

<!-- @end-section -->

<!-- @section: context-files -->
## 上下文文件

常驻上下文分两类：**AGENTS.md 分层链** 与 **persona 四件套**。

### AGENTS.md 分层加载

三个固定位置总是检查（每处都可缺席），加上向上的祖先链，按弱到强顺序注入：

1. 全局：`<home>/.stardust/agents.md`
2. 祖先链：projectRoot 之上、直到 home 的各级目录里的 `agents.md`（弱 → 强）
3. 项目：`<projectRoot>/agents.md`
4. 项目私有：`<projectRoot>/.stardust/agents.md`

每处文件名先试 `agents.md`，再试 `AGENTS.md`，两种大小写都认。`projectRoot` 与工具沙箱根（会话 `working_dir` 优先）同源——两者共用一个边界，不会出现「工具读 A 目录、规则读 B 目录」的双根问题。

配置里的 `context_files.agents_path` 与 `config_root` 已**弃用**：字段还在（兼容老的 `agent.json`），但加载器不再使用。

### persona 四件套

| 文件 | 默认路径 | 作用 |
|------|----------|------|
| `SOUL.md` | `configs/persona/SOUL.md` | 身份、风格、行为边界 |
| `TOOLS.md` | `configs/persona/TOOLS.md` | 工具使用策略 |
| `USER.md` | `configs/persona/USER.md` | 用户偏好 |
| `MEMORY.md` | `configs/persona/MEMORY.md` | 记忆快照 |

```json
{
  "context_files": {
    "enabled": true,
    "root": ".",
    "soul_path": "configs/persona/SOUL.md",
    "tools_path": "configs/persona/TOOLS.md",
    "user_path": "configs/persona/USER.md",
    "memory_path": "configs/persona/MEMORY.md",
    "max_file_chars": 20000
  }
}
```

`context_files.root` 的 `"."` 是**进程工作目录**，不是家目录也不是配置文件目录。每个文件都受 `max_file_chars` 限制。

临时关闭：

```powershell
go run ./cmd/agent -- tui --config .\agent.json --no-context-files
go run ./cmd/agent -- run --plain --config .\agent.json --prompt "..." --no-context-files
```

<!-- @end-section -->

<!-- @section: workspace -->
## workspace、docs 与 memory

```json
{
  "workspace": { "root": "~/.stardust", "docs_root": "docs", "memory_root": "memory" }
}
```

`workspace.root` 支持 `~` 展开，未设或非法时回退 `<home>/.stardust`；会话磁盘状态、docs、memory 都相对它解析。

| 类型 | 写入位置 |
|------|----------|
| 可交付说明、调研报告、runbook | `docs/` |
| 可复用经验、用户偏好、长期事实 | `memory/` |
| 启动必注入的短摘要 | `configs/persona/MEMORY.md` |
| 项目级硬规则 | `agents.md` / `AGENTS.md` |

<!-- @end-section -->

<!-- @section: tasks -->
## tasks 协作账本

```json
{
  "tasks": {
    "index_path": "tasks.md",
    "root": "tasks",
    "archive_root": "tasks/archive",
    "max_index_lines": 500,
    "max_task_lines": 300,
    "max_message_chars": 300,
    "active_statuses": ["planned", "ready", "in_progress", "blocked", "review"],
    "done_statuses": ["done", "cancelled"]
  }
}
```

`tasks/events/*.jsonl` 是唯一写入源；`tasks.md` 只是活跃索引投影，`tasks/TASK-*.md` 是单任务详情投影。长报告进 `docs/`，长期经验进 `memory/`。完整协议见 [[reference-legion-agent-tasks-md-001|tasks.md 协作规范]]。

<!-- @end-section -->

<!-- @section: subagents -->
## 子 Agent 配置

```json
{
  "agents": {
    "researcher": "configs/agents/researcher.example.json",
    "writer": "configs/agents/writer.example.json"
  }
}
```

子 Agent 通常覆盖：

| 字段 | 用途 |
|------|------|
| `id` / `role` | 标识与职责 |
| `maas_profile` | 使用哪个模型 profile |
| `context_files` | 用哪套 persona |
| `workspace` | docs/memory 输出边界 |
| `skills` | 技能目录或 registry |
| `runtime.disabled_tools` | 该 Agent 禁用哪些工具 |

未显式配置的项继承根配置。skills 同理：子 Agent 优先用自己的 `skills.install_root`，没配才回退根目录——所以 `@writer` 不会加载 `skills/researcher` 下的技能。

```json
{
  "id": "writer",
  "role": "writer",
  "maas_profile": "review",
  "skills": { "install_root": "skills/writer", "registry_url": "https://registry.example.test/index.json" },
  "runtime": { "disabled_tools": ["browser_open", "browser_click"] }
}
```

`disabled_tools` 里的名字必须在 gateable 目录里，写错名字在装配期就失败，不会静默忽略。

<!-- @end-section -->

<!-- @section: env -->
## 环境变量覆盖

| 环境变量 | 覆盖字段 |
|----------|----------|
| `LEGION_AGENT_MAAS_URL` / `LEGION_AGENT_MAAS_API_KEY` | `maas.base_url` / `maas.api_key` |
| `LEGION_AGENT_STORAGE_DRIVER` / `LEGION_AGENT_STORAGE_PATH` | `storage.*` |
| `LEGION_AGENT_SERVER_ADDR` / `LEGION_AGENT_ADMIN_TOKEN` | `server.listen_addr` / `server.admin_token` |
| `LEGION_AGENT_PUBLIC_HEALTH` | `server.public_health_enabled` |
| `LEGION_AGENT_REQUIRE_IDENTITY` | `server.require_identity`（只接受可解析布尔值，否则配置加载失败） |
| `LEGION_AGENT_REQUEST_ID_HEADER` | `server.request_id_header` |
| `LEGION_AGENT_BACKGROUND_INTERVAL` | `service.background_interval` |
| `LEGION_AGENT_DEMO_RESPONSE` / `LEGION_AGENT_MAX_TOOL_ROUNDS` | `runtime.*` |
| `LEGION_AGENT_SESSION_ENABLED` / `_RECENT_TURNS` / `_MAX_TURN_CHARS` | `session.*` |
| `LEGION_AGENT_TUI_SHOW_PROMPT` / `_SHOW_THINKING` / `_COLOR_PROFILE` | `tui.*` |
| `LEGION_AGENT_SKILL_REGISTRY_URL` / `_INSTALL_ROOT` | `skills.*` |
| `LEGION_AGENT_CONTEXT_FILES_ENABLED` / `_CONTEXT_ROOT` / `_SOUL_PATH` / `_TOOLS_PATH` / `_USER_PATH` / `_MEMORY_PATH` | `context_files.*` |
| `LEGION_AGENT_WORKSPACE_ROOT` / `_DOCS_ROOT` / `_MEMORY_ROOT` | `workspace.*` |
| `LEGION_AGENT_TASKS_INDEX_PATH` / `_ROOT` / `_ARCHIVE_ROOT` / `_MAX_INDEX_LINES` / `_MAX_TASK_LINES` / `_MAX_MESSAGE_CHARS` | `tasks.*` |
| `LEGION_AGENT_WEB_ENABLED` / `_ALLOW_PRIVATE_HOSTS` / `_TIMEOUT_SECONDS` / `_MAX_RESPONSE_KB` | `web.*` |

<!-- @end-section -->

## 相关文档

- [[reference-legion-agent-auth-001|鉴权与授权参考]] — `server.*` 安全字段与工具授权
- [[reference-legion-agent-tools-001|工具能力]] — `runtime` / `web` / `browser` 配置对应的工具行为
- [[reference-legion-agent-session-001|会话连续性]] — `session.*` 语义
- [[reference-legion-agent-tasks-md-001|tasks.md 协作规范]]
- [[agent-configuration-001|Legion Agent 配置]] — 完整字段说明
- [[reference-maas-model-profiles-001|MaaS Model Profiles]]
- [[agent-persona-files-001|运行时上下文文件]]
