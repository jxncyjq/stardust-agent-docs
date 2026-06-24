---
id: "spec-agent-evolution-evaluator-064"
title: "EvolutionEvaluator 组件规范"
aliases: ["EvolutionEvaluator规范", "进化评估器", "evolution-evaluator-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "evolution", "evaluation"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A64"
layer: "L6"
depends_on:
  - "X00"
optional_deps:
  - "A63"
conflicts_with: []
required_by: []
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# EvolutionEvaluator 组件规范

## 1. 组件定位

`EvolutionEvaluator` 衡量 Agent 能力成长，与 `TrustScoreManager` 的安全评分正交。它为管理者提供 Agent 成长报告，并为 GEP 调整提供反馈。

<!-- @end-section -->

<!-- @section: dimensions -->
---

## 2. 五维评估

| 维度 | 说明 |
|------|------|
| task_quality | 任务质量趋势 |
| learning_velocity | 学习速度 |
| reuse_effectiveness | Gene/Capsule 复用效果 |
| cost_efficiency | 成本效率 |
| stability | 行为稳定性 |

<!-- @end-section -->

<!-- @section: alerts -->
---

## 3. 退化检测

14 天窗口内同时满足以下条件才发布 `DegradationAlert`：质量下降超过阈值，且 EvalEngine 未发现外部系统性故障。

<!-- @end-section -->
