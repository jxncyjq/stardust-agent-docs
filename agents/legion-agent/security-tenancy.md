---
id: "agent-security-tenancy-001"
title: "Legion Agent Tenant 与 Company 安全边界"
type: "guide"
category: "agents/legion-agent"
tags: ["agent", "security", "tenant", "company", "p11"]
version: "0.1.0"
created: "2026-05-16"
updated: "2026-05-16"
status: "draft"
related_docs:
  - path: "../../plans/03-agent/p11-platform-integration-plan.md"
    relation: "implements"
---

# Legion Agent Tenant 与 Company 安全边界

P11 为 Agent HTTP 服务补齐 company 作用域访问控制。平台调用方可以通过请求头声明当前 company，上层资源读取时会校验资源归属。

## 请求头

| Header | 说明 |
|--------|------|
| `Authorization` | 管理令牌，格式为 `Bearer <token>` |
| `X-Company-ID` | 当前调用方所属 company |
| `X-Subject-ID` | 当前调用主体，可选 |
| `X-Role` | 当前调用角色，可选 |

## 当前保护范围

| API | 资源归属字段 | 跨 company 行为 |
|-----|--------------|-----------------|
| `GET /v1/tasks/{id}` | `Task.CompanyID` | 返回 403，并写入审计 |
| `GET /v1/workflows/{id}` | `Workflow Definition.CompanyID` | 返回 403，并写入审计 |

当前版本未提供独立 quality/audit 查询 API；后续新增这些 API 时应复用同一 company 访问判定。

## 审计行为

跨 company 访问被拒绝时写入 audit event：

| 字段 | 值 |
|------|----|
| `subject_type` | `company` |
| `subject_id` | 请求方 `X-Company-ID` |
| `action` | `access_denied.cross_company` |
| `hash` | 资源类型与资源 ID 的 SHA-256 摘要 |

## 兼容策略

未携带 `X-Company-ID` 的请求保持兼容，适用于本地 demo、smoke test 和旧调用方。生产接入时应由网关或调用方强制传入 `X-Company-ID`。

## 验证命令

```powershell
go test ./internal/server -run "TestHTTPRejectsCrossCompanyTaskAccess|TestHTTPAllowsSameCompanyTaskAccess|TestHTTPRejectsCrossCompanyWorkflowAccess" -count=1
```
