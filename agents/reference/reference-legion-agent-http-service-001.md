---
id: "reference-legion-agent-http-service-001"
title: "Legion Agent HTTP 服务"
aliases: ["agent serve", "HTTP 服务", "Agent API"]
type: "reference"
category: "agents/reference"
tags: ["agent", "http", "serve", "api", "sessions"]
version: "1.3.0"
created: "2026-05-19"
updated: "2026-05-25"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "http-api"
    relation: "related_to"
    path: "../legion-agent/http-api.md"
---

# Legion Agent HTTP 服务

## 启动

```powershell
go run ./cmd -- serve --config .\agent.json --addr :8080
```

默认监听地址来自 `server.listen_addr`，也可以用 `--addr` 覆盖。

## 常用接口

| 接口 | 说明 |
|------|------|
| `GET /healthz` | 健康检查 |
| `GET /readyz` | 依赖可用性检查 |
| `GET /metrics` | 进程内指标快照，支持 `?format=prometheus` |
| `GET /debug/diagnostics` | 脱敏诊断信息 |
| `GET /debug/traces` | 脱敏 trace 快照 |
| `GET /openapi.json` | OpenAPI 3.1 契约 |
| `GET /v1/events` | SSE 事件流 |
| `GET /v1/audit-events` | 审计事件查询 |
| `GET /v1/quality/evals` | 质量评估结果查询 |
| `POST /v1/tasks` | 提交 task |
| `GET /v1/tasks/{task_id}` | 查询 task |
| `POST /v1/workflows` | 提交 workflow definition |
| `GET /v1/workflows/{workflow_id}` | 查询 workflow 状态 |
| `GET /v1/workflows/waiting` | 查询 waiting workflow |
| `GET /v1/sessions` | 查询 session 列表 |
| `GET /v1/sessions/{session_id}/turns` | 查询会话 turns |
| `POST /v1/agents/{agent_id}/messages` | 向 Agent 发送消息 |
| `GET /v1/agents/{agent_id}/messages` | 查询 Agent inbox/outbox 消息 |

## 认证

配置了 `server.admin_token` 后，管理接口需要 Bearer token：

```powershell
curl -H "Authorization: Bearer change-me" http://127.0.0.1:8080/metrics
```

`server.public_health_enabled` 控制 `/healthz` 和 `/readyz` 是否允许匿名访问。

## RBAC 请求头

HTTP 服务使用轻量治理头表达租户和角色：

| 请求头 | 说明 |
|--------|------|
| `X-Company-ID` | 当前公司/租户 ID，不传时使用默认 company |
| `X-Subject-ID` | 当前调用主体 ID，用于审计记录 |
| `X-Role` | 角色，支持 `admin`、`operator`、`viewer` |

角色权限：

| 角色 | 能力 |
|------|------|
| `admin` 或空值 | 全部管理能力 |
| `operator` | 可读 audit、quality、task、workflow |
| `viewer` | 可读 quality、task、workflow；不能读取 audit |

被 RBAC 拒绝的请求会写入审计事件。

## OpenAPI 与错误契约

```powershell
curl http://127.0.0.1:8080/openapi.json
```

`/openapi.json` 默认公开，用于外部系统生成客户端或核对接口契约。契约包含统一 `ErrorResponse`，业务失败会返回结构化错误，而不是只返回纯文本。

## 提交 Task

```powershell
curl -X POST "http://127.0.0.1:8080/v1/tasks" `
  -H "Authorization: Bearer change-me" `
  -H "Content-Type: application/json" `
  -d "{\"id\":\"task-http-1\",\"company_id\":\"company-1\",\"agent_id\":\"cli-agent\",\"input\":\"总结当前 Agent 能力\"}"
```

查询：

```powershell
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/tasks/task-http-1"
```

## 提交 Workflow

Workflow 用于服务端多任务编排。最小示例：

```powershell
curl -X POST "http://127.0.0.1:8080/v1/workflows" `
  -H "Authorization: Bearer change-me" `
  -H "Content-Type: application/json" `
  -d "{\"id\":\"wf-quick-1\",\"root\":{\"id\":\"flow\",\"kind\":\"sequence\",\"children\":[{\"id\":\"research\",\"kind\":\"agent_task\",\"task\":{\"id\":\"research-1\",\"agent_id\":\"researcher\",\"input\":\"调研 session cache 实现\"}},{\"id\":\"write\",\"kind\":\"agent_task\",\"task\":{\"id\":\"write-1\",\"agent_id\":\"writer\",\"input\":\"根据 {{tasks.research-1.result}} 整理说明\"}}]}}"
```

查询：

```powershell
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/workflows/wf-quick-1"
```

### Workflow 与 session 历史

Workflow 不会自动读取某个 TUI session。它的上下文传递主要依赖：

- workflow 内部 `{{tasks.<task_id>.result}}` 结果占位符。
- TaskLedger `--task` 任务详情。
- AgentMessage inbox/outbox。
- 人工把 session 摘要写入 `task.input`。

如果需要让 workflow 接着某个 TUI session 的历史继续处理，先查询 session turns：

```powershell
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions/session-1770000000000000000/turns?limit=8"
```

再把摘要放进 workflow 的 `task.input`：

```json
{
  "id": "wf-from-session",
  "root": {
    "id": "writer",
    "kind": "agent_task",
    "task": {
      "id": "writer-from-session-1",
      "agent_id": "writer",
      "input": "基于 session-1770000000000000000 的历史摘要：<粘贴摘要>，继续整理说明。"
    }
  }
}
```

## Session 查询

```powershell
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions?company_id=cli-company&agent_id=cli-agent"
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions/session-123/turns?limit=6"
```

更完整的 API 说明见 [[../legion-agent/http-api|Legion Agent HTTP API]]。

## Agent 消息 API

给 writer 发送消息：

```powershell
curl -X POST "http://127.0.0.1:8080/v1/agents/writer/messages" `
  -H "Authorization: Bearer change-me" `
  -H "Content-Type: application/json" `
  -d "{\"company_id\":\"company-1\",\"from\":\"researcher\",\"task_id\":\"TASK-20260525-001\",\"type\":\"handoff\",\"summary\":\"调研完成，请整理说明\",\"artifact\":\"docs/research/session-cache.md\"}"
```

查询 writer 未读消息：

```powershell
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/agents/writer/messages?company_id=company-1&status=unread"
```

查询并标记已读：

```powershell
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/agents/writer/messages?company_id=company-1&status=unread&mark_read=true"
```

## SSE 事件流

订阅全部事件：

```powershell
curl -N -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/events"
```

按类型过滤：

```powershell
curl -N -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/events?type=task.completed"
```

事件流会对 `prompt`、`input`、`secret`、`api_key`、`token` 等敏感字段做脱敏。TUI 中的 `/event` 适合人工查看最近事件；SSE 更适合外部系统订阅。

## 治理查询

查询审计事件：

```powershell
curl -H "Authorization: Bearer change-me" `
  -H "X-Role: admin" `
  -H "X-Company-ID: company-1" `
  "http://127.0.0.1:8080/v1/audit-events"
```

查询质量评估：

```powershell
curl -H "Authorization: Bearer change-me" `
  -H "X-Role: viewer" `
  -H "X-Company-ID: company-1" `
  "http://127.0.0.1:8080/v1/quality/evals?agent_id=cli-agent"
```

`/v1/quality/evals` 支持 `agent_id`、`task_id`、`component` 过滤。

## 观测与排障

```powershell
curl -H "Authorization: Bearer change-me" http://127.0.0.1:8080/metrics
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/metrics?format=prometheus"
curl -H "Authorization: Bearer change-me" http://127.0.0.1:8080/debug/diagnostics
curl -H "Authorization: Bearer change-me" http://127.0.0.1:8080/debug/traces
```

`/metrics` 返回进程内计数指标，Prometheus 模式用于接入外部监控。`/debug/diagnostics` 和 `/debug/traces` 会输出脱敏诊断信息，用于确认配置、存储、MaaS、workflow、trace 和运行状态。
