---
id: "reference-legion-agent-config-context-001"
title: "Legion Agent 配置与上下文文件"
aliases: ["agent.json", "上下文文件", "AGENTS SOUL TOOLS USER MEMORY"]
type: "reference"
category: "agents/reference"
tags: ["agent", "configuration", "context-files", "maas", "persona"]
version: "1.2.0"
created: "2026-05-19"
updated: "2026-05-25"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "reference-configuration-001"
    relation: "related_to"
    path: "../legion-agent/configuration.md"
  - id: "reference-maas-model-profiles-001"
    relation: "related_to"
    path: "../legion-agent/model-profiles.md"
---

# Legion Agent 配置与上下文文件

## 配置文件

默认使用 `agent.json`。建议至少确认以下配置：

| 配置块 | 作用 |
|--------|------|
| `maas.default_profile` | 默认使用哪个模型 profile |
| `maas.profiles` | DeepSeek、OpenAI-compatible 或 Legion MaaS 后端配置 |
| `storage` | 是否使用 SQLite 持久化 |
| `session` | TUI 多轮会话连续性 |
| `context_files` | 是否加载 `AGENTS.md/SOUL.md/TOOLS.md/USER.md/MEMORY.md` |
| `workspace` | Agent 写文档和记忆材料的目标目录 |
| `tasks` | 多 Agent 共享任务账本路径、归档目录和膨胀控制 |
| `tui` | TUI 是否显示 prompt、thinking 和颜色主题 |

## 最小配置骨架

```json
{
  "maas": {
    "default_profile": "dev",
    "profiles": {}
  },
  "storage": {
    "driver": "sqlite",
    "path": "agent.db"
  },
  "server": {
    "listen_addr": ":8080",
    "admin_token": "change-me",
    "public_health_enabled": true
  },
  "runtime": {
    "max_tool_rounds": 4
  },
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

`storage.driver=sqlite` 是推荐配置。只有 SQLite 模式才会持久化 task、event、audit、session、message 等数据；`memory` 模式适合临时 demo。

## MaaS 配置

OpenAI-compatible 配置示例：

```json
{
  "maas": {
    "default_profile": "dev",
    "profiles": {
      "dev": {
        "base_url": "https://api.example.com",
        "model": "example-chat-model",
        "api_key": "change-me"
      }
    }
  }
}
```

当 `model` 非空时，Agent 会调用 `{base_url}/chat/completions`。当 `model` 为空时，Agent 会按 Legion 内部 MaaS 协议调用 `{base_url}/v1/inference/generate`。

Legion 内部 MaaS 协议配置示例：

```json
{
  "maas": {
    "default_profile": "legion-maas",
    "profiles": {
      "legion-maas": {
        "base_url": "http://127.0.0.1:9000",
        "api_key": "change-me"
      }
    }
  }
}
```

选择 profile 的优先级：

| 来源 | 说明 |
|------|------|
| `--maas-profile` | 命令行显式指定，优先级最高 |
| 子 Agent `maas_profile` | `@researcher`、workflow `agent_id` 命中子 Agent 时使用 |
| `maas.default_profile` | 默认 profile |
| `--maas-url` | 临时覆盖 base URL，并绕过 profile client 构建 |

API key 不建议写入可提交的配置文件。生产和共享环境应通过安全配置、CI/CD secret 或本地未提交文件管理。

可用环境变量覆盖常用字段：

| 环境变量 | 覆盖字段 |
|----------|----------|
| `LEGION_AGENT_MAAS_URL` | `maas.base_url` |
| `LEGION_AGENT_MAAS_API_KEY` | `maas.api_key` |
| `LEGION_AGENT_STORAGE_DRIVER` | `storage.driver` |
| `LEGION_AGENT_STORAGE_PATH` | `storage.path` |
| `LEGION_AGENT_SERVER_ADDR` | `server.listen_addr` |
| `LEGION_AGENT_ADMIN_TOKEN` | `server.admin_token` |
| `LEGION_AGENT_TUI_COLOR_PROFILE` | `tui.color_profile` |

## session cache 配置

`session` 支持短期上下文窗口 cache：

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

`cache_enabled` 只影响最近 conversation turns 的进程内缓存，不影响 `agent_sessions`、`conversation_turns` 的 SQLite 持久化。`cache_max_entries` 控制最多缓存多少个 session context window。

## 上下文文件

Agent 启动任务时会按配置加载五类上下文文件：

| 文件 | 默认路径 | 作用 |
|------|----------|------|
| `AGENTS.md` | `AGENTS.md` | 项目规则、构建验证、文档约定 |
| `SOUL.md` | `configs/persona/SOUL.md` | Agent 身份、风格和行为边界 |
| `TOOLS.md` | `configs/persona/TOOLS.md` | 工具使用策略和禁止伪工具输出 |
| `USER.md` | `configs/persona/USER.md` | 用户偏好 |
| `MEMORY.md` | `configs/persona/MEMORY.md` | 记忆快照 |

加载顺序是固定的：项目规则、人格、工具策略、用户偏好、记忆快照。加载结果会作为 Runtime context files 注入 prompt。每个文件都会受 `max_file_chars` 限制，避免超大文件挤占上下文窗口。

临时关闭上下文文件：

```powershell
go run ./cmd -- tui --config .\agent.json --no-context-files
go run ./cmd -- run --plain --config .\agent.json --prompt "..." --no-context-files
```

## docs 与 memory

`docs/` 和 `memory/` 是运行时输出目录约定：

| 目录 | 用途 |
|------|------|
| `docs/` | 可读文档、报告、runbook、决策记录 |
| `memory/` | 后续可检索、可归并的记忆材料 |

`configs/persona/USER.md` 和 `configs/persona/MEMORY.md` 是启动时注入的快照；`memory/` 保存更细粒度的材料，后续可以人工归并回快照文件。

建议约定：

| 类型 | 写入位置 |
|------|----------|
| 可交付说明、调研报告、runbook | `docs/` |
| 可复用经验、用户偏好、长期事实 | `memory/` |
| 启动时必须注入的短摘要 | `configs/persona/MEMORY.md` |
| 项目级硬规则 | `AGENTS.md` |

## tasks 协作账本

`tasks` 配置约定多 Agent 交换任务上下文的位置：

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

`tasks/events/*.jsonl` 是协作账本的唯一写入源；`tasks.md` 只保留活跃任务索引和最新摘要；`tasks/TASK-*.md` 是单任务详情投影。长报告写入 `docs/`；长期经验写入 `memory/`。普通 Agent 应通过 TaskLedger 工具追加事件，不直接覆盖投影文件。完整协议见 [[reference-legion-agent-tasks-md-001|tasks.md 协作规范]]。

## 子 Agent 配置

根配置通过 `agents` 注册子 Agent：

```json
{
  "agents": {
    "researcher": "configs/agents/researcher.example.json",
    "writer": "configs/agents/writer.example.json"
  }
}
```

子 Agent 通常覆盖这些字段：

| 字段 | 用途 |
|------|------|
| `agent.id` / `agent.role` | 标识和职责 |
| `maas_profile` | 使用哪个模型 profile |
| `context_files` | 使用哪个 SOUL/MEMORY/TOOLS/USER |
| `workspace` | docs/memory 输出边界 |
| `skills` | 技能安装目录或 registry |

未显式配置的共享项会继承根配置。`skills` 也跟随这个规则：主 Agent 使用根配置 `skills.install_root`；子 Agent 优先使用自己的 `skills.install_root`；子 Agent 没有配置时回退到根配置。

示例：

```json
{
  "id": "writer",
  "role": "writer",
  "maas_profile": "review",
  "skills": {
    "install_root": "skills/writer",
    "registry_url": "https://registry.example.test/index.json"
  }
}
```

运行时会只从当前 Agent 的 skills root 挂载匹配技能到 prompt 中。也就是说 `@writer` 不会加载 `skills/researcher` 里的技能，除非 writer 未配置自己的 `skills.install_root` 并继承了根目录。
