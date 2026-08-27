---
id: "agent-governance-rbac-001"
title: "Legion Agent Governance RBAC"
type: "guide"
category: "backend/agent"
tags: ["agent", "governance", "rbac", "security"]
version: "1.1.0"
created: "2026-06-24"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "agent-index-001"
    relation: "related_to"
    path: "./index.md"
  - id: "reference-legion-agent-auth-001"
    relation: "related_to"
    path: "../reference/reference-legion-agent-auth-001.md"
---

# Legion Agent Governance RBAC

P12 为治理查询接口增加 role + company 边界，覆盖审计事件和质量评估历史。该能力用于企业控制台、运维排障和质量治理看板，避免只依赖 admin token 造成过宽读取权限。

## 身份头

| Header | 说明 |
|--------|------|
| `Authorization: Bearer <token>` | 服务管理令牌，仍是 HTTP 管理接口的第一层保护 |
| `X-Company-ID` | 当前请求所属 company，用于资源作用域判断 |
| `X-Subject-ID` | 当前主体 ID，可用于后续审计扩展 |
| `X-Role` | 当前主体角色，支持 `admin`、`operator`、`viewer` |

为兼容本地开发和既有脚本，缺失 `X-Role` 时按 `admin` 处理。生产接入应显式传入 role。

## 权限矩阵

| Role | audit | quality | task | workflow |
|------|-------|---------|------|----------|
| `admin` | read | read | read | read |
| `operator` | read | read | read | read |
| `viewer` | denied | read | read | read |
| unknown | denied | denied | denied | denied |

## 接口

```http
GET /v1/audit-events
GET /v1/quality/evals?agent_id=agent-1&task_id=task-1&component=planner
```

`/v1/audit-events` 需要 `admin` 或 `operator`。`viewer` 访问会返回 `403`，并写入审计动作：

```text
access_denied.rbac
```

`/v1/quality/evals` 允许 `viewer` 读取质量评估历史，支持按 `agent_id`、`task_id`、`component` 过滤。

## 当前边界

- task/workflow 继续使用已有 company 资源校验。
- audit 事件当前领域模型尚未携带独立 `company_id` 字段，因此 P12-001 先完成 role 边界与拒绝审计。
- quality eval 当前按 Agent/Task/Component 过滤；后续如引入 Agent-company 映射，应在查询层增加 company join/filter。

