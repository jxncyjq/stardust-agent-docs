---
id: "spec-agent-tool-guardrails-023"
title: "ToolGuardrails 组件规范"
aliases: ["ToolGuardrails规范", "工具看门狗", "tool-guardrails-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "tool", "guardrails", "watchdog"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A23"
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
# ToolGuardrails 组件规范

## 1. 组件定位

`ToolGuardrails` 是工具调用前后的看门狗，负责重复失败检测、危险参数检查、结果异常识别与循环风险信号输出。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
type ToolGuardrails interface {
    BeforeCall(ctx ToolGuardContext, call ToolCall) (GuardDecision, error)
    AfterCall(ctx ToolGuardContext, call ToolCall, result ToolResult) (GuardDecision, error)
}

type GuardDecision struct {
    Action GuardAction // allow | warn | block | mark_anomaly
    Reason string
    Signal *LearningSignal
}
```

<!-- @end-section -->

<!-- @section: rules -->
---

## 3. 内置规则

| 规则 | 阶段 | 动作 |
|------|------|------|
| 同一工具同一参数连续失败 ≥3 次 | after | `mark_anomaly` |
| 输出为空但退出码成功连续 ≥3 次 | after | `warn` |
| 参数含绝对外部路径 | before | `block` |
| 高风险命令未获批准 | before | `block` |
| 工具返回疑似 prompt injection | after | `warn` 并降低信任分 |

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[agent-runtime-spec|AgentRuntime 组件规范]]
- [[eval-engine-spec|EvalEngine 组件规范]]

<!-- @end-section -->
