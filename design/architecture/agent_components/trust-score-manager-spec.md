---
id: "spec-agent-trust-score-manager-061"
title: "TrustScoreManager 组件规范"
aliases: ["TrustScoreManager规范", "信任评分", "trust-score-manager-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "trust", "security"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A61"
layer: "L6"
depends_on:
  - "X00"
  - "X02"
optional_deps: []
conflicts_with: []
required_by:
  - "A02"
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# TrustScoreManager 组件规范

## 1. 组件定位

`TrustScoreManager` 衡量 Agent 的安全行为可信度，与 `EvolutionEvaluator` 的能力成长评分正交。它影响派遣、工具权限和高风险任务准入。

<!-- @end-section -->

<!-- @section: scoring -->
---

## 2. 评分规则

| 项 | 默认 |
|----|------|
| 初始分 | 0.7 |
| 范围 | 0.0~1.0 |
| 冷却期 | 严重事件触发，懒惰求值 |
| 低分阻断 | `<0.3` 阻止执行高风险任务 |

事件示例：权限拒绝、注入命中、秘密暴露、HardLoop、人工恢复成功、连续安全完成。

<!-- @end-section -->

<!-- @section: interface -->
---

## 3. 接口定义

```go
type TrustScoreManager interface {
    LogSecurityEvent(ctx context.Context, event SecurityEvent) error
    EffectiveScore(ctx context.Context, agentID string, at time.Time) (float64, error)
    CanExecute(ctx context.Context, agentID string, risk RiskLevel) (TrustDecision, error)
}
```

<!-- @end-section -->
