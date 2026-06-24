---
id: "reference-legion-agent-troubleshooting-001"
title: "Legion Agent 常见问题排查"
aliases: ["Agent 排障", "troubleshooting", "TUI 问题"]
type: "reference"
category: "agents/reference"
tags: ["agent", "troubleshooting", "tui", "maas", "session"]
version: "1.1.0"
created: "2026-05-19"
updated: "2026-05-25"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
---

# Legion Agent 常见问题排查

## TUI 没有进入交互

请使用 `tui` 子命令，而不是 `run`：

```powershell
go run ./cmd -- tui --config .\agent.json
```

`run` 是单次任务模式，任务结束后会退出。

## 模型没有被调用

检查顺序：

1. `maas.default_profile` 是否指向存在的 profile。
2. profile 是否配置了 `base_url`。
3. OpenAI-compatible 接口是否配置了非空 `model`。
4. 是否被命令行 `--maas-url` 覆盖。
5. 日志文件 `logs/agent.log` 是否有调用错误。

快速验证配置是否真的走 HTTP MaaS：

```powershell
go run ./cmd -- run --plain --config .\agent.json --prompt "只回答 ok"
```

如果返回 demo 文本或固定文本，检查是否误用了 `--demo`，或配置中没有可用 `base_url`。如果报 400，请优先看上游模型服务返回的错误正文，常见原因是 `model` 名不对、工具 schema 不被上游兼容、API key 无效。

## OpenAI-compatible 返回 400

常见原因：

| 现象 | 处理 |
|------|------|
| `Invalid schema for function ...` | 工具参数 schema 不符合上游要求，先升级当前代码或临时减少工具调用 |
| `model not found` | 检查 `maas.profiles.<name>.model` |
| `unauthorized` | 检查 `api_key` 或 `LEGION_AGENT_MAAS_API_KEY` |
| `404 /chat/completions` | base URL 不应包含 `/chat/completions`，只写服务根地址 |

## TUI 颜色异常

降低颜色能力：

```powershell
$env:LEGION_AGENT_TUI_COLOR_PROFILE = "ansi256"
go run ./cmd -- tui --config .\agent.json
```

也可以在配置中设置：

```json
{
  "tui": {
    "color_profile": "ansi256"
  }
}
```

## 上下文没有连续

确认：

1. `storage.driver` 是 `sqlite`。
2. `session.enabled` 是 `true`。
3. `session.default_recent_turns` 大于 0。
4. 没有执行 `/new` 或 `/clear-session` 清空当前上下文。

如果刚修改过配置，重启 TUI。session cache 是进程内缓存，追加新 turn 或切换 session 会自动失效；如果仍怀疑缓存影响调试，可以临时设置：

```json
{
  "session": {
    "cache_enabled": false
  }
}
```

## `@researcher` 或 `@writer` 不可用

检查：

1. `agent.json` 是否配置了 `agents`。
2. 子 Agent 配置文件路径是否相对于 `agent.json` 所在目录可解析。
3. TUI 输入 `@` 是否能弹出候选。
4. 子 Agent 的 `maas_profile` 是否存在。

最小配置：

```json
{
  "agents": {
    "researcher": "configs/agents/researcher.example.json",
    "writer": "configs/agents/writer.example.json"
  }
}
```

## `/send` 有消息但 `@writer --inbox` 没读到

检查：

1. `/send writer ...` 的目标 Agent ID 是否和 `@writer` 完全一致。
2. 是否已经被上一次 `@writer --inbox` 成功处理并标记 read。
3. 是否切换了数据库或使用了 `storage.driver=memory`。
4. HTTP 查询时 `company_id` 是否一致。

## 工具输出看起来像伪调用

Agent 已有输出净化策略会拦截 `search_content(...)`、`read_file(...)` 等伪工具文本。若模型仍回答“不确定”或要求先搜索，说明当前任务未经过真实工具协议执行，应使用已接入的内置工具能力或在后续 P21/P工具批次继续补齐。

## 运行日志在哪里

TUI/CLI 默认不把运行日志直接打到界面，日志追加到：

```text
logs/agent.log
```

排查模型调用、工具调用和运行时错误时，优先查看该文件。

PowerShell 查看最近日志：

```powershell
Get-Content .\logs\agent.log -Tail 80
```

## 数据库在哪里

默认 SQLite 文件是：

```text
agent.db
```

如果配置了 `storage.path`，以配置为准。备份和恢复：

```powershell
go run ./cmd -- backup --config .\agent.json --out .\backups\agent.db.bak
go run ./cmd -- restore --config .\agent.json --in .\backups\agent.db.bak
```
