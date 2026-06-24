---
id: "spec-agent-signal-extractor-050"
title: "SignalExtractor 组件规范"
aliases: ["SignalExtractor规范", "信号提取器", "signal-extractor-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "learning", "gep", "signal"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A50"
layer: "L5"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "A51"
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# SignalExtractor 组件规范

## 1. 组件定位

`SignalExtractor` 从任务轨迹、失败事件、人工反馈和 Eval 指标中提取学习信号，是 GEP 周期的入口。

<!-- @end-section -->

<!-- @section: signal-levels -->
---

## 2. 三层信号

| 层级 | 来源 | 示例 |
|------|------|------|
| L1 正则/结构化 | 错误码、状态机、工具结果 | `HardLoopFailure`、`BudgetExhausted` |
| L2 关键词/模式 | 日志、人工反馈、拒绝理由 | “重复尝试同一工具” |
| L3 LLM 归纳 | 多任务轨迹综合 | “缺少先验边界检查策略” |

<!-- @end-section -->

<!-- @section: behavior -->
---

## 3. 行为契约

- 8 个周期内同类低价值信号出现 ≥3 次则抑制。
- HardLoop、权限越权、秘密泄露永不抑制。
- 每个信号必须包含 source、evidence、confidence。
- 输出只描述问题与机会，不直接生成 Gene。

<!-- @end-section -->
