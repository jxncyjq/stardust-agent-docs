---
id: "spec-agent-execution-policy-021"
title: "ExecutionPolicy 组件规范"
aliases: ["ExecutionPolicy规范", "执行策略", "批准策略引擎"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "approval", "sandbox", "rbac"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A21"
layer: "L2"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "A20"
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# ExecutionPolicy 组件规范

## 1. 组件定位

`ExecutionPolicy` 将批准策略与沙盒模式正交组合，决定一次工具调用是否可自动执行、需要人工批准、或直接拒绝。

<!-- @end-section -->

<!-- @section: policy -->
---

## 2. 策略模型

| 维度 | 可选值 | 说明 |
|------|--------|------|
| ApprovalPolicy | `auto_allow` / `ask` / `deny` | 是否需要人类批准 |
| SandboxMode | `read_only` / `workspace_write` / `full_access` | 文件与进程隔离边界 |
| RBAC Override | role → policy | 不同 Agent 角色默认策略不同 |
| AuditAll | bool | 是否记录所有工具调用 |

<!-- @end-section -->

<!-- @section: interface -->
---

## 3. 接口定义

```go
type ExecutionPolicy interface {
    Decide(ctx PolicyContext, call ToolCall) (PolicyDecision, error)
}

type PolicyDecision struct {
    Action      PolicyAction // allow | ask | deny
    Sandbox     SandboxMode
    Reason      string
    ExpiresAt   *time.Time
}
```

<!-- @end-section -->

<!-- @section: rules -->
---

## 4. 关键规则

- `deny` 优先级高于任何 allowlist。
- `auto_allow` 只允许精确工具名或受限前缀，不允许裸 `*`。
- 写文件、网络、启动进程默认至少 `ask`。
- 批准结果必须带有效期和理由，写入审计链。

<!-- @end-section -->
