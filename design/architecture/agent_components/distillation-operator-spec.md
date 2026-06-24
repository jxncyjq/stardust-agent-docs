---
id: "spec-agent-distillation-operator-052"
title: "DistillationOperator 组件规范"
aliases: ["DistillationOperator规范", "蒸馏算子", "strategy-gene"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "learning", "gene", "distillation"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A52"
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
# DistillationOperator 组件规范

## 1. 组件定位

`DistillationOperator` 将单次轨迹 `s` 或历史轨迹集合 `H` 蒸馏为 Strategy Gene，目标是把经验压缩为可注入、可审核、可评估的能力资产。

<!-- @end-section -->

<!-- @section: gene-format -->
---

## 2. Gene 六元组

| 字段 | 含义 | 约束 |
|------|------|------|
| `m` | match 条件 | 明确适用任务边界 |
| `u` | compact summary | 约 50 tokens |
| `pi` | procedure intuition | 简短策略步骤 |
| `alpha` | AVOID 线索 | 强制非空 |
| `c` | confidence | 0~1 |
| `v` | version | 内容寻址版本 |

单个 Gene token 预算默认 ≤ 230。

<!-- @end-section -->

<!-- @section: operators -->
---

## 3. 蒸馏算子

| 算子 | 输入 | 输出 |
|------|------|------|
| `psi(s)` | 单次成功/失败轨迹 | 候选 Gene |
| `psi(H)` | 多个相似历史轨迹 | 归纳 Gene |
| `refine(g, feedback)` | Gene + 人工反馈 | 修订 Gene |

<!-- @end-section -->
