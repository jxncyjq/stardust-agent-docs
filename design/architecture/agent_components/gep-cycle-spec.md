---
id: "spec-agent-gep-cycle-051"
title: "GepCycle 组件规范"
aliases: ["GepCycle规范", "GEP周期", "gep-cycle-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "learning", "gep"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A51"
layer: "L5"
depends_on:
  - "A50"
  - "A52"
  - "A53"
  - "A54"
  - "A43"
optional_deps: []
conflicts_with: []
required_by: []
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# GepCycle 组件规范

## 1. 组件定位

`GepCycle` 编排 GEP 六阶段学习周期，把失败/成功信号转为可审核的 Gene 或代码固化变更。

<!-- @end-section -->

<!-- @section: stages -->
---

## 2. 六阶段流程

| 阶段 | 输入 | 输出 |
|------|------|------|
| scan | LearningEvent / EvalRun | 候选轨迹集合 |
| signal | 轨迹集合 | LearningSignal |
| intent | 信号 | 变更意图 |
| mutate | 意图 | Gene 草稿或代码补丁草案 |
| validate | 草稿 | 验证结果 |
| solidify | 通过验证的草稿 | 固化资产 |

<!-- @end-section -->

<!-- @section: gates -->
---

## 3. 门控规则

- `validate` 前可配置人工审核门控。
- Critical 安全信号不得自动固化。
- 每轮只处理一个主焦点，避免多目标突变。
- 所有阶段快照写入 `EvolutionEventLog`。

<!-- @end-section -->
