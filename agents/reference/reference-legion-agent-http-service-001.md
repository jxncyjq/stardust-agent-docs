---
id: "reference-legion-agent-http-service-001"
title: "Legion Agent HTTP 服务"
aliases: ["agent serve", "HTTP 服务", "Agent API"]
type: "reference"
category: "agents/reference"
tags: ["agent", "http", "serve", "api", "sessions"]
version: "2.0.0"
created: "2026-05-19"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "reference-legion-agent-backend-api-001"
    relation: "extends"
    path: "./reference-legion-agent-backend-api-001.md"
  - id: "reference-legion-agent-auth-001"
    relation: "depends_on"
    path: "./reference-legion-agent-auth-001.md"
  - id: "agent-http-api-001"
    relation: "related_to"
    path: "../legion-agent/http-api.md"
---

# Legion Agent HTTP 服务

本文是**使用者视角的 curl 速查**。端点全表、请求体字段、错误码口径见 [[reference-legion-agent-backend-api-001|后端系统调用参考]]；鉴权规则见 [[reference-legion-agent-auth-001|鉴权与授权参考]]。

## 启动

```powershell
go run ./cmd/agent -- serve --config .\agent.json --addr :8080
```

默认监听地址来自 `server.listen_addr`（缺省 `:8080`），`--addr` 覆盖。**不配 `listen_addr` 也不传 `--addr`** 时绑到 `127.0.0.1:0`（随机端口，GUI 内嵌形态），并自动进入 loopback 加固。

验证：

```powershell
curl http://127.0.0.1:8080/healthz
curl http://127.0.0.1:8080/readyz
```

## 常用接口速查

| 接口 | 说明 |
|------|------|
| `GET /healthz` `GET /readyz` | 存活 / 就绪 |
| `GET /metrics` | 指标快照，`?format=prometheus` |
| `GET /debug/diagnostics` `GET /debug/traces` | 脱敏诊断与 trace |
| `GET /openapi.json` | OpenAPI 3.1 契约（恒公开） |
| `GET /v1/events` | 平台事件 SSE |
| `POST /v1/tasks` `GET /v1/tasks` `GET /v1/tasks/{id}` `GET /v1/tasks/{id}/result` | 任务提交、列表、状态、结果 |
| `POST /v1/tasks/{id}/interrupt` | 中断运行中的任务 |
| `GET /v1/approvals` `POST /v1/tasks/{id}/approvals/{ticketID}` | Manual 模式审批 |
| `POST/GET /v1/sessions`、`PATCH/DELETE /v1/sessions/{id}`、`GET /v1/sessions/{id}/turns` | 会话生命周期与历史 |
| `GET /v1/agents`、`GET/POST /v1/agents/{id}/messages` | 子 Agent 名单与消息 |
| `POST /v1/workflows`、`GET /v1/workflows/{id}`、`POST /v1/workflows/{id}/events`、`GET /v1/workflows/waiting` | workflow |
| `GET /v1/audit-events` `GET /v1/runtime-events` `GET /v1/quality/evals` | 治理与观测查询 |
| `GET /v1/plugins`、`POST /v1/plugins/{name}/grant`、`POST /v1/plugins/{name}/deny` | 插件同意流 |
| `POST /v1/skills/install`、`/v1/skills/update`、`/v1/skills/uninstall` | 技能管理 |
| `GET /v1/files` | 下载/预览会话工作目录内文件 |
| `/v1/browser/sessions/{id}/` 下的 `stream`、`takeover`、`viewport`、`input` | 内置浏览器流与接管 |

## 认证

配置 `server.admin_token` 后，除 `/openapi.json`（恒公开）和 `public_health_enabled=true` 时的 `/healthz`、`/readyz` 外，全部端点要求 Bearer：

```powershell
curl -H "Authorization: Bearer change-me" http://127.0.0.1:8080/metrics
```

`admin_token` 为空 = 全部端点无鉴权放行，只适合纯本机。loopback 加固模式下 token 每次启动重铸并写进握手文件，见 [[reference-legion-agent-auth-001|鉴权与授权参考]]。

## RBAC 请求头

| 请求头 | 说明 |
|--------|------|
| `X-Company-ID` | 租户 ID |
| `X-Subject-ID` | 调用主体，用于审计 |
| `X-Role` | `admin` / `operator` / `viewer` |

| 角色 | 能力 |
|------|------|
| `admin`（`require_identity=false` 时空角色等同 admin） | 全部，含插件 grant/deny |
| `operator` | 读 audit、quality、task、workflow、plugin |
| `viewer` | 读 quality、task、workflow；不能读 audit / plugin |

`server.require_identity=true` 后缺身份即拒。被拒的跨公司访问会写审计事件 `access_denied.cross_company`。

## 提交与查询 Task

```powershell
curl -X POST "http://127.0.0.1:8080/v1/tasks" `
  -H "Authorization: Bearer change-me" `
  -H "Content-Type: application/json" `
  -d "{\"id\":\"task-http-1\",\"company_id\":\"company-1\",\"agent_id\":\"cli-agent\",\"input\":\"总结当前 Agent 能力\"}"

curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/tasks/task-http-1"
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/tasks/task-http-1/result"
```

`/result` 除答案文本外还返回 token 用量（prompt / completion / cached / total）、耗时和 `generated_files` 文件链接。

中断长任务：

```powershell
curl -X POST -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/tasks/task-http-1/interrupt"
```

## 会话

```powershell
# 新建（id 由服务端生成）
curl -X POST "http://127.0.0.1:8080/v1/sessions" `
  -H "Authorization: Bearer change-me" -H "Content-Type: application/json" `
  -d "{\"company_id\":\"cli-company\",\"agent_id\":\"cli-agent\",\"title\":\"接口验证\",\"mode\":\"auto\",\"working_dir\":\"F:/work/demo\"}"

# 列表 / turns
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions?company_id=cli-company&agent_id=cli-agent"
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions/session-123/turns?limit=6"

# 改标题 / 归档 / 切换工作模式
curl -X PATCH "http://127.0.0.1:8080/v1/sessions/session-123" `
  -H "Authorization: Bearer change-me" -H "Content-Type: application/json" `
  -d "{\"title\":\"改个名\",\"mode\":\"manual\"}"

# 删除（同时删除该会话的磁盘目录）
curl -X DELETE -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions/session-123"
```

`working_dir` 一旦设定就不能改指向别的目录（改会 400）；未知字段会被 400 拒绝而不是静默忽略。

## 提交 Workflow

```powershell
curl -X POST "http://127.0.0.1:8080/v1/workflows" `
  -H "Authorization: Bearer change-me" `
  -H "Content-Type: application/json" `
  -d "{\"id\":\"wf-quick-1\",\"root\":{\"id\":\"flow\",\"kind\":\"sequence\",\"children\":[{\"id\":\"research\",\"kind\":\"agent_task\",\"task\":{\"id\":\"research-1\",\"agent_id\":\"researcher\",\"input\":\"调研 session cache 实现\"}},{\"id\":\"write\",\"kind\":\"agent_task\",\"task\":{\"id\":\"write-1\",\"agent_id\":\"writer\",\"input\":\"根据 {{tasks.research-1.result}} 整理说明\"}}]}}"

curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/workflows/wf-quick-1"
```

Workflow 不会自动挂载某个 TUI session。要让它接着历史对话工作，先取 turns 再把摘要写进 `task.input`：

```powershell
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions/session-1770000000000000000/turns?limit=8"
```

## Agent 消息 API

```powershell
# 发送
curl -X POST "http://127.0.0.1:8080/v1/agents/writer/messages" `
  -H "Authorization: Bearer change-me" -H "Content-Type: application/json" `
  -d "{\"company_id\":\"company-1\",\"from\":\"researcher\",\"task_id\":\"TASK-20260525-001\",\"type\":\"handoff\",\"summary\":\"调研完成，请整理说明\",\"artifact\":\"docs/research/session-cache.md\"}"

# 查询未读 / 查询并标记已读
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/agents/writer/messages?company_id=company-1&status=unread"
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/agents/writer/messages?company_id=company-1&status=unread&mark_read=true"
```

查询子 Agent 名单（GUI 选择器用的就是它）：

```powershell
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/agents"
```

## SSE 事件流

```powershell
curl -N -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/events"
curl -N -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/events?type=task_completed"
```

事件类型与 `RuntimeEvent.Type` 同名：`task_started`、`inference_completed`、`tool_call_requested`、`tool_executed`、`tool_failed`、`tool_result`、`tool_loop_broken`、`subtask_completed`、`task_completed`、`task_cancelled`。`prompt`、`input`、`secret`、`api_key`、`token` 等敏感键出站前剔除，单个字符串超 512 字符截断。

不想开长连接时用 `GET /v1/runtime-events` 拉最近 200 条。

## 生成文件下载

```powershell
curl -H "Authorization: Bearer change-me" `
  "http://127.0.0.1:8080/v1/files?session_id=session-123&path=docs%2Freport.md&download=1" -o report.md
```

链接直接用 `/v1/tasks/{id}/result` 返回的 `url` / `download_url`；服务端只存工作区相对路径，链接每次现拼，改 `server.file_base_url` 立即生效。路径越出会话工作目录返回 403。

## 治理查询

```powershell
curl -H "Authorization: Bearer change-me" -H "X-Role: admin" -H "X-Company-ID: company-1" `
  "http://127.0.0.1:8080/v1/audit-events"

curl -H "Authorization: Bearer change-me" -H "X-Role: viewer" -H "X-Company-ID: company-1" `
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

每个响应都带 `X-Request-ID`（请求带同名头则透传），日志里可以按它对齐一次请求。

## 相关文档

- [[reference-legion-agent-backend-api-001|后端系统调用参考]] — 端点全表、请求体、错误码
- [[reference-legion-agent-auth-001|鉴权与授权参考]] — token / 握手 / RBAC
- [[reference-legion-agent-integration-001|接入参考]] — 客户端接入形态
- [[reference-legion-agent-session-001|会话连续性]]
- [[agent-http-api-001|Legion Agent HTTP API]]
