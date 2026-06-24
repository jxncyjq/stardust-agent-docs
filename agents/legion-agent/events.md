---
id: "agent-events-001"
title: "Legion Agent 平台事件流"
type: "guide"
category: "agents/legion-agent"
tags: ["agent", "events", "sse", "eventbus", "p11"]
version: "0.1.0"
created: "2026-05-15"
updated: "2026-05-15"
status: "draft"
related_docs:
  - path: "../../design/architecture/common_components/event-bus-spec.md"
    relation: "implements"
  - path: "../../plans/03-agent/p11-platform-integration-plan.md"
    relation: "implements"
---

# Legion Agent 平台事件流

P11 引入平台级事件流，用于让控制台、前端和外部运维系统订阅 Agent 运行状态。HTTP 入口为 `/v1/events`，采用 Server-Sent Events。

## 订阅方式

```http
GET /v1/events?type=task.completed
Authorization: Bearer <admin-token>
```

`type` 可选；不传时订阅全部平台事件，传入时只返回匹配类型。

## SSE 格式

```text
event: task.completed
: subject_id=task-1
data: {"task_id":"task-1"}
```

## 事件信封

| 字段 | 说明 |
|------|------|
| `id` | 可选事件 ID |
| `type` | 事件类型，如 `task.completed`、`workflow.completed`、`quality.degraded` |
| `subject_id` | 可选资源 ID |
| `data` | 事件摘要 payload |
| `created_at` | 事件创建时间 |

## 脱敏规则

SSE 写出前会移除敏感 payload 字段：

| 字段 | 处理 |
|------|------|
| `prompt` | 删除 |
| `input` | 删除 |
| `secret` | 删除 |
| `api_key` | 删除 |
| 包含 `secret` 或 `token` 的字段 | 删除 |

## 验证命令

```powershell
go test ./internal/server -run TestSSEEventsFiltersAndSanitizesPayload -count=1
```
