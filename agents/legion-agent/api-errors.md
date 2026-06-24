---
id: "agent-api-errors-001"
title: "Legion Agent API Error Contract"
type: "guide"
category: "backend/agent"
tags: ["agent", "api", "error", "contract"]
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

# Legion Agent API Error Contract

P12 为 OpenAPI 增加统一错误响应矩阵，避免客户端只依赖成功响应而忽略治理、认证和服务异常路径。

## ErrorResponse

所有受保护接口的错误响应统一使用：

```json
{
  "error": "unauthorized"
}
```

OpenAPI schema 固定为：

```json
{
  "type": "object",
  "required": ["error"],
  "properties": {
    "error": {
      "type": "string"
    }
  }
}
```

## 错误码矩阵

受 `AdminToken` 保护的接口统一声明以下错误响应：

| HTTP status | 含义 |
|-------------|------|
| `400` | 请求体、参数或事件内容不合法 |
| `401` | 缺少或错误的 `Authorization: Bearer <admin-token>` |
| `403` | 已认证但 role/company 边界不允许访问 |
| `500` | 内部依赖或持久化异常 |

当前覆盖的主要接口包括：

- `/metrics`
- `/debug/diagnostics`
- `/debug/traces`
- `/v1/audit-events`
- `/v1/quality/evals`
- `/v1/events`
- `/v1/tasks`
- `/v1/tasks/{id}`
- `/v1/workflows`
- `/v1/workflows/{id}`
- `/v1/workflows/{id}/events`
- `/v1/workflows/waiting`

## 兼容性门禁

- `internal/server/openapi_test.go` 检查核心路径和受保护接口错误响应矩阵。
- `internal/compat/openapi_golden_test.go` 锁定完整 `openapi-agent.json`。
- `internal/compat/openapi_error_golden_test.go` 单独锁定 `ErrorResponse` schema，便于客户端 SDK 或平台网关快速识别破坏性变更。
