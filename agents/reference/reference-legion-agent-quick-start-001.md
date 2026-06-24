---
id: "reference-legion-agent-quick-start-001"
title: "Legion Agent 快速开始"
aliases: ["Agent 快速开始", "quick start", "启动 Agent"]
type: "reference"
category: "agents/reference"
tags: ["agent", "quick-start", "startup", "doctor"]
version: "1.1.0"
created: "2026-05-19"
updated: "2026-05-25"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
---

# Legion Agent 快速开始

这篇文档用于把 Agent 在本机跑起来。首次使用建议按顺序完成：进入目录、确认配置、启动 TUI、发第一条消息、查看日志和数据库。

## 项目目录

在 Legion 主仓库中，Agent 代码目录为：

```text
legion/legionAgent
```

常用目录：

| 路径 | 用途 |
|------|------|
| `agent.json` | 本地运行配置 |
| `configs/persona/` | `SOUL.md`、`TOOLS.md`、`USER.md`、`MEMORY.md` 默认人格与上下文文件 |
| `configs/agents/` | `researcher`、`writer` 等子 Agent 示例配置 |
| `docs/` | Agent 运行时写文档的默认目标目录 |
| `memory/` | Agent 运行时输出记忆材料的默认目录 |
| `logs/agent.log` | TUI/CLI 默认运行日志 |
| `agent.db` | SQLite 持久化数据库 |

## 启动前检查

进入 Agent 项目目录：

```powershell
cd F:\source\stardust\Legion\legion\legionAgent
```

检查本地环境和版本：

```powershell
go run ./cmd -- doctor
go run ./cmd -- version --plain
```

确认配置文件存在：

```powershell
Test-Path .\agent.json
```

如果还没有可用配置，可以先复制完整样例再按本机模型服务修改：

```powershell
Copy-Item .\configs\agent.complete.example.json .\agent.json
```

最小可用配置需要包含 SQLite 存储、MaaS profile、上下文文件和 TUI 设置。OpenAI-compatible 模型服务通常这样写：

```json
{
  "maas": {
    "default_profile": "dev",
    "profiles": {
      "dev": {
        "base_url": "https://api.example.com",
        "model": "deepseek-chat",
        "api_key": "change-me"
      }
    }
  },
  "storage": {
    "driver": "sqlite",
    "path": "agent.db"
  },
  "session": {
    "enabled": true,
    "default_recent_turns": 6,
    "max_turn_chars": 6000,
    "restore_latest_on_tui_start": true,
    "cache_enabled": true,
    "cache_max_entries": 128
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
  }
}
```

如果不想把 API key 写入文件，可以用环境变量或命令行参数临时覆盖：

```powershell
$env:LEGION_AGENT_MAAS_API_KEY = "sk-..."
go run ./cmd -- tui --config .\agent.json
```

## 三种运行方式

交互式 TUI：

```powershell
go run ./cmd -- tui --config .\agent.json
```

单次非交互任务：

```powershell
go run ./cmd -- run --plain --config .\agent.json --prompt "总结当前 Agent 的能力"
```

HTTP 服务：

```powershell
go run ./cmd -- serve --config .\agent.json --addr :8080
```

## 第一次 TUI 对话

启动 TUI 后，在底部 Composer 输入：

```text
你是什么模型？当前可以使用哪些工具？
```

按 Enter 发送。正常情况下你会看到：

- 主输出区出现你的问题和模型回复。
- 底部状态栏显示工作中进度条，完成后回到可输入状态。
- `logs/agent.log` 记录运行日志。
- `agent.db` 中写入 task、event、audit、session turn 等数据。

继续输入第二轮问题：

```text
基于上一轮回答，说明 session 是如何保持上下文的。
```

如果 `session.enabled=true` 且使用 SQLite，第二轮会自动带入最近对话上下文。

## 第一次多 Agent 调用

确认 `agent.json` 中注册了子 Agent：

```json
{
  "agents": {
    "researcher": "configs/agents/researcher.example.json",
    "writer": "configs/agents/writer.example.json"
  }
}
```

在 TUI 中输入 `@` 会弹出已注册 Agent。可以直接委托：

```text
@researcher 调研当前 session/cache 实现
@writer 整理成一份面向用户的说明
```

需要显式交接消息时：

```text
/send writer researcher 已完成调研，请整理成说明
@writer --inbox 根据未读消息继续处理
```

## 本地验证

```powershell
go test ./...
go vet ./...
go build -o NUL ./cmd
```

日常只验证 TUI/配置/session 时，可以先跑较小范围：

```powershell
go test ./internal/cli ./internal/tui ./internal/config ./internal/sessioncache -count=1
```

## 常见下一步

| 目标 | 下一篇文档 |
|------|------------|
| 配置 MaaS、上下文文件和 docs/memory | [[reference-legion-agent-config-context-001\|配置与上下文文件]] |
| 熟悉 TUI 快捷键、slash 命令、`@agent` | [[reference-legion-agent-tui-001\|TUI 使用]] |
| 理解多轮对话为什么连续 | [[reference-legion-agent-session-001\|会话连续性]] |
| 使用 researcher/writer 协作 | [[reference-legion-agent-multi-agent-usage-001\|多 Agent 调用]] |
| 启动服务和调用 HTTP API | [[reference-legion-agent-http-service-001\|HTTP 服务]] |
