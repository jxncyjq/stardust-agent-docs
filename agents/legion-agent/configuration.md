---
id: "agent-configuration-001"
title: "Legion Agent 配置"
type: "guide"
category: "backend/agent"
tags: ["agent", "configuration", "config"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "agent-index-001"
    relation: "related_to"
    path: "./index.md"
---

# Legion Agent 配置

Legion Agent 支持 JSON 配置文件，并允许环境变量覆盖关键运行参数。

## 示例

完整示例文件位于 `legion/legionAgent/configs/agent.full.example.json`，可复制为本地 `agent.json` 后替换 token、API key 和路径。示例中的 `_comment_*` 字段用于在合法 JSON 中承载注释，当前配置加载器会忽略这些字段。

```json
{
  "agents": {
    "researcher": "configs/agents/researcher.example.json",
    "writer": "configs/agents/writer.example.json"
  },
  "maas": {
    "base_url": "https://maas.example.test",
    "api_key": "change-me",
    "default_profile": "fast",
    "profiles": {
      "fast": {
        "base_url": "https://maas-fast.example.test",
        "model": "",
        "api_key": "change-me-fast"
      },
      "review": {
        "base_url": "https://maas-review.example.test",
        "model": "",
        "api_key": "change-me-review"
      }
    }
  },
  "storage": {
    "driver": "sqlite",
    "path": "agent.db"
  },
  "server": {
    "listen_addr": ":8080",
    "admin_token": "change-me",
    "public_health_enabled": true,
    "request_id_header": "X-Request-ID"
  },
  "service": {
    "background_interval": "1s"
  },
  "runtime": {
    "demo_response": "task completed",
    "max_tool_rounds": 4
  },
  "tui": {
    "show_prompt": true,
    "show_thinking": true,
    "color_profile": "truecolor"
  },
  "context_files": {
    "enabled": true,
    "root": ".",
    "agents_path": "AGENTS.md",
    "soul_path": "configs/persona/SOUL.md",
    "tools_path": "configs/persona/TOOLS.md",
    "user_path": "configs/persona/USER.md",
    "memory_path": "configs/persona/MEMORY.md",
    "max_file_chars": 20000
  },
  "workspace": {
    "docs_root": "docs",
    "memory_root": "memory"
  },
  "tasks": {
    "index_path": "tasks.md",
    "root": "tasks",
    "archive_root": "tasks/archive",
    "max_index_lines": 500,
    "max_task_lines": 300,
    "max_message_chars": 300,
    "active_statuses": ["planned", "ready", "in_progress", "blocked", "review"],
    "done_statuses": ["done", "cancelled"]
  },
  "skills": {
    "registry_url": "https://registry.example.test/index.json",
    "install_root": "skills"
  }
}
```

## 多 Agent 子配置

根配置的 `agents` 字段把 workflow `task.agent_id` 映射到子 Agent 配置文件：

```json
{
  "agents": {
    "researcher": "configs/agents/researcher.example.json",
    "writer": "configs/agents/writer.example.json"
  }
}
```

子 Agent 配置只描述差异项：`id`、`role`、`maas_profile`、`context_files`、`workspace`、`skills`。MaaS endpoint、storage、server、runtime max tool rounds 等共享项继承根配置。

`agent serve` 会在执行 workflow `agent_task` 时保留 `task.agent_id`，并按子配置切换 role、MaaS profile、上下文文件和只读工具 workspace。未配置或未命中的 `agent_id` 会回退到默认 Agent runtime。

## CLI 使用

```powershell
go run ./cmd -- run --plain --config .\agent.json --prompt "Summarize Legion Agent"
go run ./cmd -- serve --config .\agent.json --addr :8080
go run ./cmd -- tui --config .\agent.json
```

命令行参数优先于配置文件。例如同时传入 `--maas-url` 和配置文件时，会使用命令行的 MaaS URL。

`agent run` 默认会按 `context_files` 加载 `AGENTS.md/SOUL.md/TOOLS.md/USER.md/MEMORY.md` 并注入 MaaS prompt。临时关闭可传 `--no-context-files`。

如果要接 DeepSeek/OpenAI-compatible API，请在 profile 中配置 `model`，例如：

```json
{
  "maas": {
    "default_profile": "dev",
    "profiles": {
      "dev": {
        "base_url": "https://api.deepseek.com",
        "model": "deepseek-chat",
        "api_key": "change-me"
      }
    }
  }
}
```

配置了 `model` 的 profile 会请求 `base_url + /chat/completions`；未配置 `model` 时使用 Legion 内部 MaaS endpoint `/v1/inference/generate`。

`agent tui` 是交互式 Bubble Tea 界面，启动后可输入 prompt 并按 Enter 执行任务。它与 `agent run` 使用同一套 `context_files` 配置，因此默认也会加载 `AGENTS.md/SOUL.md/TOOLS.md/USER.md/MEMORY.md`。

TUI 主会话区默认会先显示用户输入的问题，再显示 `thinking`，最后显示模型输出。`thinking` 会优先显示 MaaS/OpenAI-compatible 响应中明确返回的公开 reasoning 字段，例如内部 MaaS 的 `reasoning_summary` 或兼容 Chat Completions 消息中的 `reasoning_content`；如果上游没有返回公开 reasoning 字段，则回落为 Agent 可观测流程摘要，例如接收 prompt、准备上下文、通过 C70 MaaS 端口调用模型、等待输出和收到事件。若需要更简洁的界面，可关闭对应字段：

```json
{
  "tui": {
    "show_prompt": false,
    "show_thinking": false
  }
}
```

### TUI 配色方案

`tui.color_profile` 控制 Bubble Tea TUI 使用的终端颜色能力，与 [termenv](https://github.com/muesli/termenv) 的 Profile 概念对应：

| 值 | 别名 | 颜色位数 | 适用场景 |
|---|---|---|---|
| `truecolor` | `24bit` | 24 位真彩色（默认） | Windows Terminal、iTerm2、现代 VTE 终端 |
| `ansi256` | `256` | 256 色 | xterm-256color、SSH 到旧服务器 |
| `ansi` | `16` | 16 色 ANSI | 传统终端、PuTTY、某些 SSH 客户端 |
| `ascii` | `none` / `no-color` | 无颜色 | CI 日志、管道输出、屏幕阅读器 |

若不填，默认为 `truecolor`。也可通过环境变量覆盖：

```powershell
$env:LEGION_AGENT_TUI_COLOR_PROFILE = "ansi256"
go run ./cmd -- tui --config .\agent.json
```

> **提示**：如果 TUI 颜色显示异常（如颜色方块乱码），说明终端不支持 `truecolor`，请改为 `ansi256` 或 `ansi`。

当 `storage.driver` 为 `sqlite` 时，CLI run 和 service HTTP API 会把 task、runtime event、audit、waiting workflow state 写入 `storage.path` 指定的数据库文件。

## HTTP 管理面安全

`server.admin_token` 用于保护除公开健康探针之外的 HTTP 管理接口。为空时保持本地开发模式开放；生产环境必须设置非空 token。

```powershell
curl -H "Authorization: Bearer change-me" http://127.0.0.1:8080/v1/workflows/waiting
```

`server.public_health_enabled` 控制 `/healthz` 是否允许匿名访问。`server.request_id_header` 控制请求追踪响应头，默认是 `X-Request-ID`；客户端传入该 header 时服务会原样回传，否则服务端生成一个新的 request id。

## 环境变量覆盖

| 环境变量 | 配置字段 |
|----------|----------|
| `LEGION_AGENT_MAAS_URL` | `maas.base_url` |
| `LEGION_AGENT_MAAS_API_KEY` | `maas.api_key` |
| `LEGION_AGENT_STORAGE_DRIVER` | `storage.driver` |
| `LEGION_AGENT_STORAGE_PATH` | `storage.path` |
| `LEGION_AGENT_SERVER_ADDR` | `server.listen_addr` |
| `LEGION_AGENT_ADMIN_TOKEN` | `server.admin_token` |
| `LEGION_AGENT_PUBLIC_HEALTH` | `server.public_health_enabled` |
| `LEGION_AGENT_REQUEST_ID_HEADER` | `server.request_id_header` |
| `LEGION_AGENT_BACKGROUND_INTERVAL` | `service.background_interval` |
| `LEGION_AGENT_DEMO_RESPONSE` | `runtime.demo_response` |
| `LEGION_AGENT_MAX_TOOL_ROUNDS` | `runtime.max_tool_rounds` |
| `LEGION_AGENT_TUI_SHOW_PROMPT` | `tui.show_prompt` |
| `LEGION_AGENT_TUI_SHOW_THINKING` | `tui.show_thinking` |
| `LEGION_AGENT_TUI_COLOR_PROFILE` | `tui.color_profile` |
| `LEGION_AGENT_SKILL_REGISTRY_URL` | `skills.registry_url` |
| `LEGION_AGENT_SKILL_INSTALL_ROOT` | `skills.install_root` |
| `LEGION_AGENT_CONTEXT_FILES_ENABLED` | `context_files.enabled` |
| `LEGION_AGENT_CONTEXT_ROOT` | `context_files.root` |
| `LEGION_AGENT_AGENTS_PATH` | `context_files.agents_path` |
| `LEGION_AGENT_SOUL_PATH` | `context_files.soul_path` |
| `LEGION_AGENT_TOOLS_PATH` | `context_files.tools_path` |
| `LEGION_AGENT_USER_PATH` | `context_files.user_path` |
| `LEGION_AGENT_MEMORY_PATH` | `context_files.memory_path` |
| `LEGION_AGENT_DOCS_ROOT` | `workspace.docs_root` |
| `LEGION_AGENT_MEMORY_ROOT` | `workspace.memory_root` |
| `LEGION_AGENT_TASKS_INDEX_PATH` | `tasks.index_path` |
| `LEGION_AGENT_TASKS_ROOT` | `tasks.root` |
| `LEGION_AGENT_TASKS_ARCHIVE_ROOT` | `tasks.archive_root` |
| `LEGION_AGENT_TASKS_MAX_INDEX_LINES` | `tasks.max_index_lines` |
| `LEGION_AGENT_TASKS_MAX_TASK_LINES` | `tasks.max_task_lines` |
| `LEGION_AGENT_TASKS_MAX_MESSAGE_CHARS` | `tasks.max_message_chars` |

## 字段说明

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `maas.base_url` | 空 | 旧版单 MaaS endpoint。未配置 profile 时使用 |
| `agents` | `{}` | 多 Agent 子配置映射，key 对应 workflow `task.agent_id`，value 为相对根配置文件目录的 JSON 路径 |
| `maas.api_key` | 空 | 旧版单 endpoint API key |
| `maas.default_profile` | 空 | 未传 `--maas-profile` 时默认使用的 profile |
| `maas.profiles` | `{}` | 多模型 profile 映射，供 `agent run --maas-profile <name>` 选择 |
| `maas.profiles.<name>.model` | 空 | 可选。非空时使用 OpenAI-compatible chat completions 协议 |
| `storage.driver` | `memory` | 支持 `memory` 或 `sqlite` |
| `storage.path` | `agent.db` | SQLite 数据库路径。若使用子目录，需先创建父目录 |
| `server.listen_addr` | `:8080` | `agent serve` 监听地址 |
| `server.admin_token` | 空 | HTTP 管理接口 token。生产环境必须设置 |
| `server.public_health_enabled` | `true` | 是否允许匿名访问 `/healthz`、`/readyz` |
| `server.request_id_header` | `X-Request-ID` | 请求追踪 header 名称 |
| `service.background_interval` | `1s` | 后台调度间隔，使用 Go duration 格式 |
| `runtime.demo_response` | `task completed` | 未配置真实 MaaS 时的 demo 响应 |
| `runtime.max_tool_rounds` | `4` | 模型工具调用最大闭环轮数，超过后报错防止无限循环 |
| `tui.show_prompt` | `true` | 是否在 TUI 主会话区显示用户刚输入的问题 |
| `tui.show_thinking` | `true` | 是否在 TUI 主会话区显示 thinking；优先为模型服务返回的公开 reasoning，否则为 Agent 运行过程摘要 |
| `tui.color_profile` | `truecolor` | TUI 配色方案：`truecolor`、`ansi256`、`ansi`、`ascii`，详见 [TUI 配色方案](#tui-配色方案) |
| `context_files.enabled` | `true` | 是否加载运行时上下文文件 |
| `context_files.root` | `.` | 上下文文件允许读取的根目录 |
| `context_files.agents_path` | `AGENTS.md` | 项目规则文件路径 |
| `context_files.soul_path` | `configs/persona/SOUL.md` | Agent 身份文件路径 |
| `context_files.tools_path` | `configs/persona/TOOLS.md` | 工具策略文件路径 |
| `context_files.user_path` | `configs/persona/USER.md` | 用户偏好快照路径 |
| `context_files.memory_path` | `configs/persona/MEMORY.md` | Agent 记忆快照路径 |
| `context_files.max_file_chars` | `20000` | 单个上下文文件最大注入字符数 |
| `workspace.docs_root` | `docs` | Agent 运行时文档输出目录 |
| `workspace.memory_root` | `memory` | Agent 运行时记忆材料输出目录 |
| `tasks.index_path` | `tasks.md` | 多 Agent 共享任务索引投影，只保留活跃任务摘要，由 TaskLedger 重建 |
| `tasks.root` | `tasks` | 任务账本目录；`events/*.jsonl` 为事件源，`TASK-*.md` 为单任务详情投影 |
| `tasks.archive_root` | `tasks/archive` | 完成或取消任务的归档投影目录 |
| `tasks.max_index_lines` | `500` | `tasks.md` 最大建议行数，超过后应归档或拆分 |
| `tasks.max_task_lines` | `300` | 单个任务详情文件最大建议行数 |
| `tasks.max_message_chars` | `300` | 写入索引的单条消息摘要最大建议字符数 |
| `tasks.active_statuses` | `planned, ready, in_progress, blocked, review` | 保留在任务索引里的活跃状态 |
| `tasks.done_statuses` | `done, cancelled` | 应从索引移入归档的终态 |
| `skills.registry_url` | 空 | 远端 Skill registry index URL |
| `skills.install_root` | `skills` | 技能安装目录 |
