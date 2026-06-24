---
id: "spec-agent-permission-enforcer-022"
title: "PermissionEnforcer 组件规范"
aliases: ["PermissionEnforcer规范", "权限执行器", "permission-enforcer-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "permission", "security"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A22"
layer: "L2"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "A01"
  - "A20"
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# PermissionEnforcer 组件规范

## 1. 组件定位

`PermissionEnforcer` 在工具执行前执行权限检查，覆盖文件路径、网络域名、进程启动、凭证访问、组织数据访问等资源边界。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
type PermissionEnforcer interface {
    Check(ctx PermissionContext, request PermissionRequest) (PermissionDecision, error)
    CheckBatch(ctx PermissionContext, requests []PermissionRequest) ([]PermissionDecision, error)
}

type PermissionDecision struct {
    Allowed bool
    Reason  string
    RuleID  string
}
```

<!-- @end-section -->

<!-- @section: resource-types -->
---

## 3. 资源类型

| 类型 | 示例 | 默认 |
|------|------|------|
| filesystem | `docs/**`, `src/**` | workspace 内可写，外部需审批 |
| network | `api.openai.com` | deny |
| process | `go test`, `bun run build` | ask |
| secret | API Key / token | deny |
| org_data | tenant/team/agent records | RBAC 决定 |

<!-- @end-section -->

<!-- @section: contracts -->
---

## 4. 行为契约

- 支持批量检查，避免 N 次工具调用产生 N 次 DB 查询。
- 拒绝结果必须包含命中的规则 ID。
- 路径匹配必须使用规范化绝对路径，防止 `..` 绕过。
- 所有拒绝与人工覆盖必须写入审计链。

<!-- @end-section -->
