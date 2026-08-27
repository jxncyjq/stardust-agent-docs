---
id: "reference-legion-agent-troubleshooting-001"
title: "Legion Agent 常见问题排查"
aliases: ["Agent 排障", "troubleshooting", "TUI 问题"]
type: "reference"
category: "agents/reference"
tags: ["agent", "troubleshooting", "tui", "maas", "session"]
version: "2.0.0"
created: "2026-05-19"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "reference-legion-agent-tools-001"
    relation: "related_to"
    path: "./reference-legion-agent-tools-001.md"
  - id: "reference-legion-agent-auth-001"
    relation: "related_to"
    path: "./reference-legion-agent-auth-001.md"
  - id: "reference-legion-agent-backend-api-001"
    relation: "related_to"
    path: "./reference-legion-agent-backend-api-001.md"
---

# Legion Agent 常见问题排查

## TUI 没有进入交互

请使用 `tui` 子命令，而不是 `run`：

```powershell
go run ./cmd/agent -- tui --config .\agent.json
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
go run ./cmd/agent -- run --plain --config .\agent.json --prompt "只回答 ok"
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
go run ./cmd/agent -- tui --config .\agent.json
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

输出净化会拦截 `search_content(...)`、`read_file(...)` 这类伪工具文本。若模型仍说「需要先搜索」，说明它没有返回结构化 `tool_calls`：先确认模型服务支持 function calling，再看 `/event` 里有没有 `tool_call_requested`。

## 工具相关

| 现象 | 处理 |
|------|------|
| 某工具在能力清单里根本不存在 | `web_search` 需要 `web.searxng_url` 非空才注册；`browser_*` 需要 `browser.enabled=true`；也可能被 `runtime.disabled_tools` 禁了 |
| 工具报 `permission denied` | 该工具没登记进角色允许表；新增工具必须同步登记，enforcer 跑在策略之前 |
| 启动报「未知工具名」 | `runtime.disabled_tools` 里的名字必须是 gateable 目录里的工具 |
| 同一个工具被反复调用后中断 | 触发单工具 30 次上限，发 `tool_loop_broken` 事件；查为什么模型在原地打转（多半是上一轮结果没被利用） |
| 任务报超过工具轮数 | `runtime.max_tool_rounds` 默认 4，按需调大 |
| Manual 模式任务一直挂着 | 没人裁决审批：拉 `GET /v1/approvals`，用 `POST /v1/tasks/{id}/approvals/{ticketID}` 批；超时（默认 300 秒）会自动按拒绝返回 |
| 文件路径被拒 | 路径必须落在该任务 ToolRoot 内（会话 `working_dir` 优先，否则配置根） |
| `generated_files` 为空 | `write_file` 默认 `overwrite=false`，目标文件已存在时写入失败被跳过——验证请换新文件名 |

## HTTP 接入相关

| 现象 | 处理 |
|------|------|
| 401 unauthorized | 配了 `admin_token` 却没带 Bearer；GUI/loopback 模式下 token 每次启动重铸，必须现读 |
| 403 forbidden origin | loopback 加固下 `Origin` 与服务 baseURL 不一致 |
| 403 company access denied | 开了 `require_identity` 却没注入 `X-Company-ID` |
| 404 not found（插件端点） | 该进程没配 `plugins.manifest`，不等于「没有插件」 |
| 404（中断任务） | 任务已结束或不存在，契约上不会把「没在跑」报成成功 |
| 400 unknown field | 会话相关端点拒收未知字段（如客户端自带 `id`） |
| 任务卡在 pending | 并发已满（`runtime.max_concurrent_tasks` 默认 4），或调度未运行，看 `/debug/diagnostics` |

## prompt 越来越大 / token 消耗异常

单任务 input 涨到 40-60k 通常**不是 bug**：多轮工具循环每轮都会重发累积的会话。排查顺序：

1. 打开 `runtime.debug`，它会在每次推理前按消息打印角色、字符数、工具调用数和预览，定位是哪条消息在膨胀。
2. 看轮数：能用一次检索解决的别让模型翻十次文件。
3. 需要时打开 `runtime.compact_token_threshold` 触发会话压缩（单任务最多压 3 次）。
4. 对账用 `audit_events`（按时间窗查），`conversation_turns` 的 token 列只对新会话有值，老数据是 0。

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
go run ./cmd/agent -- backup --config .\agent.json --out .\backups\agent.db.bak
go run ./cmd/agent -- restore --config .\agent.json --in .\backups\agent.db.bak
```

## 相关文档

- [[reference-legion-agent-tools-001|工具能力]] — 工具清单、执行管线与上限
- [[reference-legion-agent-auth-001|鉴权与授权参考]] — 401/403 的完整判定规则
- [[reference-legion-agent-backend-api-001|后端系统调用参考]] — 错误码口径
- [[reference-legion-agent-config-context-001|配置与上下文文件]] — 相关配置字段
- [[reference-legion-agent-session-001|会话连续性]] — 上下文不连续的排查
