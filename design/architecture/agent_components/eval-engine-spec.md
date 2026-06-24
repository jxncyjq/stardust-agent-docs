---
id: "spec-agent-eval-engine-063"
title: "EvalEngine 组件规范"
aliases: ["EvalEngine规范", "四层Eval", "eval-engine-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "eval", "behavior-health"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A63"
layer: "L6"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "A01"
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# EvalEngine 组件规范

## 1. 组件定位

`EvalEngine` 提供 Agent 行为健康评估，覆盖实时循环检测与后台趋势评估，是 HardLoop 风险控制的核心输入。

<!-- @end-section -->

<!-- @section: layers -->
---

## 2. 四层 Eval

| 层 | 名称 | 窗口 | 用途 |
|----|------|------|------|
| L1 | Output Quality | 单任务 | 输出格式、完成度 |
| L2 | Trace Health | 当前任务/24h | 收敛比、重复工具调用 |
| L3 | Component Health | 14d | 工具、技能、记忆质量 |
| L4 | Drift Detection | 8周趋势 | 行为漂移和退化 |

实时 `eval_trace()` 使用当前任务窗口，不使用历史 24h 窗口触发 HardLoop。

<!-- @end-section -->

<!-- @section: thresholds -->
---

## 3. 循环阈值

| 信号 | 条件 | 动作 |
|------|------|------|
| SoftLoop | convergence ratio > 5 | 触发 ContextCompressor checkpoint |
| HardLoop | convergence ratio > 10 | 返回 `AnomalyDetected`，进入审批链 |

<!-- @end-section -->
