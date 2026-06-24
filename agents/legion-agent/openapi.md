---
id: "agent-openapi-001"
title: "Legion Agent OpenAPI 契约"
type: "guide"
category: "agents/legion-agent"
tags: ["agent", "openapi", "api", "compatibility", "p11"]
version: "0.1.0"
created: "2026-05-15"
updated: "2026-05-15"
status: "draft"
related_docs:
  - path: "../../plans/03-agent/p11-platform-integration-plan.md"
    relation: "implements"
---

# Legion Agent OpenAPI 契约

Agent 服务提供 `/openapi.json` 作为平台集成入口，格式为 OpenAPI 3.1 JSON。该接口不要求管理令牌，便于控制台、SDK 生成器和兼容性检查发现 API。

## 当前覆盖范围

| 路径 | 说明 | 鉴权 |
|------|------|------|
| `/healthz` | 存活检查 | 公开 |
| `/readyz` | 依赖就绪检查 | 可配置公开 |
| `/metrics` | 指标快照 | 管理令牌 |
| `/debug/diagnostics` | 诊断摘要 | 管理令牌 |
| `/openapi.json` | API 契约 | 公开 |
| `/v1/tasks` | 提交任务 | 管理令牌 |
| `/v1/tasks/{id}` | 查询任务 | 管理令牌 |
| `/v1/workflows` | 提交工作流 | 管理令牌 |
| `/v1/workflows/waiting` | 查询等待中的工作流 | 管理令牌 |
| `/v1/workflows/{id}` | 查询工作流状态 | 管理令牌 |
| `/v1/workflows/{id}/events` | 投递工作流恢复事件 | 管理令牌 |
| `/v1/events` | 平台事件订阅 | 管理令牌 |

## 兼容性门禁

OpenAPI 输出由 `internal/compat/testdata/openapi-agent.json` 固化，并由 `TestOpenAPIGolden` 检查。新增路径、重命名 operationId、删除 schema 名称、改变安全方案时，都需要显式更新 golden。

## 敏感信息规则

OpenAPI schema 只描述平台对象，不包含 prompt、tool input、secret、api key 的示例或默认值。诊断和事件输出也必须只暴露摘要字段。

## 验证命令

```powershell
go test ./internal/server ./internal/compat -run "TestOpenAPISpecIncludesCorePlatformRoutes|TestHTTPServerServesOpenAPIWithoutAdminToken|TestOpenAPIGolden" -count=1
```
