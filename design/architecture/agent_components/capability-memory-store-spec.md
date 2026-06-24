---
id: "spec-agent-capability-memory-store-043"
title: "CapabilityMemoryStore 组件规范"
aliases: ["CapabilityMemoryStore规范", "能力记忆库", "Gene资产库"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "memory", "gene", "capsule"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A43"
layer: "L4"
depends_on: []
optional_deps:
  - "X01"
conflicts_with: []
required_by:
  - "A51"
  - "A53"
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# CapabilityMemoryStore 组件规范

## 1. 组件定位

`CapabilityMemoryStore` 保存经 GEP 固化的能力资产，包括 Strategy Gene、Capsule 与可复用行为模式。它是学习进化层与认知内核之间的能力资产边界。

<!-- @end-section -->

<!-- @section: assets -->
---

## 2. 资产类型

| 类型 | 说明 | 注入规则 |
|------|------|----------|
| Gene | 六元组 `m/u/pi/alpha/c/v`，约 230 tokens | 最多 3 个/任务 |
| Capsule | 多 Gene 组合和验证记录 | 只在高相似任务注入 |
| Avoid Hint | 失败规避线索 | `alpha` 字段强制非空 |

<!-- @end-section -->

<!-- @section: interface -->
---

## 3. 接口定义

```go
type CapabilityMemoryStore interface {
    PutGene(ctx context.Context, gene Gene) error
    SearchGenes(ctx context.Context, query CapabilityQuery) ([]GeneHit, error)
    MarkOutcome(ctx context.Context, geneID string, outcome CapabilityOutcome) error
    PromoteCapsule(ctx context.Context, capsule Capsule) error
}
```

<!-- @end-section -->

<!-- @section: governance -->
---

## 4. 治理规则

- 未通过验证的 Gene 状态为 `draft`，不得注入。
- 管理员可禁用或删除有害 Gene。
- 成功率、最近失败、blast radius 共同决定排序。
- 每次注入写入审计，便于评估 Gene 是否真的提升任务质量。

<!-- @end-section -->
