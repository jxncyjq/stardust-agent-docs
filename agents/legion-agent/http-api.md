---
id: "agent-http-api-001"
title: "Legion Agent HTTP API"
type: "api"
category: "backend/agent"
tags: ["agent", "http", "api"]
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

# Legion Agent HTTP API

`agent serve` 会启动 HTTP API。默认监听地址来自配置 `server.listen_addr`，也可以用命令行覆盖。

```powershell
go run ./cmd -- serve --config .\agent.json --addr :8080
```

## Endpoints

| Method | Path | 说明 |
|--------|------|------|
| `GET` | `/healthz` | 健康检查，返回 `{"status":"ok"}` |
| `GET` | `/readyz` | 依赖可用性检查，返回 storage readiness |
| `GET` | `/metrics` | 返回内存指标快照，受 `server.admin_token` 保护 |
| `GET` | `/debug/diagnostics` | 返回脱敏诊断快照，受 `server.admin_token` 保护 |
| `POST` | `/v1/tasks` | 提交 pending task |
| `GET` | `/v1/tasks/{task_id}` | 查询 task 状态 |
| `GET` | `/v1/workflows/waiting` | 列出 waiting workflow 状态 |
| `GET` | `/v1/sessions` | 查询 Agent 会话列表，支持 `company_id`、`agent_id` 过滤 |
| `GET` | `/v1/sessions/{session_id}/turns` | 查询指定会话的 conversation turns，支持 `limit` |

## Submit Task

```json
{
  "id": "task-1",
  "company_id": "company-1",
  "agent_id": "agent-1",
  "input": "Summarize Legion Agent"
}
```

当前 P8 HTTP API 先提供服务面和状态面，任务执行调度与真实持久化运行模式将在后续 P8 任务继续接入。

## Sessions

P22 起，TUI 多轮对话会写入 `agent_sessions` 与 `conversation_turns`。`agent serve` 在 SQLite 存储模式下会把同一份 session store 暴露为只读 HTTP 查询接口。

```powershell
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions?company_id=cli-company&agent_id=cli-agent"
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions/session-123/turns?limit=6"
```

`/v1/sessions` 返回 `AgentSession` 数组，按最近更新时间倒序排列。`/v1/sessions/{session_id}/turns` 返回 `ConversationTurn` 数组，`limit=0` 或省略表示返回该 session 的全部 turns。

## Metrics

`GET /metrics` 返回当前进程内的运行指标快照。配置了 `server.admin_token` 时，需要带上 Bearer token。

```powershell
curl -H "Authorization: Bearer change-me" http://127.0.0.1:8080/metrics
```

响应示例：

```json
{
  "started_at": "2026-05-14T08:30:00Z",
  "tasks": {
    "submitted": 2,
    "running": 1,
    "done": 1
  },
  "http_status": {
    "200": 8,
    "201": 2,
    "401": 1
  },
  "model_calls": {
    "success": 1
  },
  "approvals": {},
  "workflow_runs": {
    "waiting": 1
  }
}
```

当前指标为单进程内存快照，用于本地诊断、smoke 和早期运维观测；进程重启后会重新计数。

## Readiness

`GET /readyz` 用于检查服务依赖是否可用。当前会检查 storage；当 `storage.driver=sqlite` 时，会调用 SQLite 轻量 ping。

可用时：

```json
{
  "status": "ok",
  "checks": {
    "storage": "ok"
  }
}
```

不可用时返回 `503`：

```json
{
  "status": "unavailable",
  "reason": "storage_unavailable",
  "checks": {
    "storage": "unavailable"
  }
}
```

`/readyz` 与 `/healthz` 一样受 `server.public_health_enabled` 控制；关闭公开探针后需要 Bearer token。

## Diagnostics

`GET /debug/diagnostics` 返回当前进程诊断快照，用于排障和运维交接。该接口始终属于管理接口；配置了 `server.admin_token` 时必须携带 Bearer token。

```powershell
curl -H "Authorization: Bearer change-me" http://127.0.0.1:8080/debug/diagnostics
```

诊断快照包含：

| 字段 | 说明 |
|------|------|
| `version` / `commit` / `build_time` | 构建版本信息 |
| `uptime_seconds` | 当前进程运行秒数 |
| `config` | 脱敏配置轮廓 |
| `scheduler` | 后台调度启用与运行状态 |
| `metrics` | 与 `/metrics` 相同的内存指标快照 |

诊断快照不会输出 MaaS API key、admin token、完整 prompt 或完整 storage path；敏感字段显示为 `[redacted]`，storage path 只输出 hash。
