---
id: "spec-agent-aegis-reviewer-060"
title: "AegisReviewer 组件规范"
aliases: ["AegisReviewer规范", "Aegis质量评审器", "aegis-reviewer-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "quality", "aegis"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A60"
layer: "L6"
depends_on:
  - "C70"
  - "X00"
optional_deps: []
conflicts_with: []
required_by: []
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# AegisReviewer 组件规范

## 1. 组件定位

`AegisReviewer` 是固定 LLM 质量评审器。它通过 `C70 MaasInferenceClient` 调用 MaaS 层，不走 `AgentRuntime`，不产生 Gene，也不需要被自身审核。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
type AegisReviewer interface {
    Review(ctx context.Context, task Task, artifact Artifact, criteria ReviewCriteria) (ReviewResult, error)
}
```

<!-- @end-section -->

<!-- @section: rules -->
---

## 3. 审核规则

| 项 | 默认 |
|----|------|
| 模型等级 | Advanced Tier |
| 最大驳回轮次 | 3 |
| 输出格式 | `VERDICT: APPROVED/REJECTED` + reasons |
| 预算策略 | 不受低余额预算降级影响，但仍计费 |

只有 `APPROVED` 可驱动 `QualityReview → Done`。

## 4. MaaS 调用约束

- 使用 `MaasInferenceClient.Complete(PurposeAegisReview)`。
- 不直接调用 `ModelRouter`、`ModelProvider` 或 provider SDK。
- 评审请求必须携带 `task_id`、`artifact_id`、`criteria_id`，便于 MaaS 审计和平台证据链关联。

<!-- @end-section -->
