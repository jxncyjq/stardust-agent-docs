---
id: "plans-agent-learning-quality-001"
title: "Agent 学习进化与质量治理计划"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "memory", "learning", "quality"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# Agent 学习进化与质量治理计划

## 目标

让 Agent 从任务反馈、反思、协作观察和知识库中学习，同时避免学习到绕过安全边界的错误经验。

## 组件计划

| 能力 | 组件 | 交付 |
|------|------|------|
| 工作记忆 | A41 | 单任务草稿本和中间状态 |
| 情景记忆 | A42/X01 | 历史任务经验向量检索 |
| 能力记忆 | A43 | Gene/Capsule 能力资产 |
| 信号提取 | A50 | failure/success/hardloop/feedback 信号 |
| GEP 周期 | A51/A52/A53/A54 | scan、mutate、validate、solidify、audit |
| 质量评估 | A60/A63/A64 | Aegis、Eval、退化检测 |
| 信任治理 | A61/A62 | trust score、审批门控 |

## 风险控制

- 学习资产写入前必须经过安全扫描。
- 低置信记忆只作为候选上下文，不作为硬规则。
- 质量退化触发冻结或回滚建议。
- 人类可删除、冻结、重置 Agent 记忆和能力资产。

