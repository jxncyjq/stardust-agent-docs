---
id: "plans-agent-runtime-mvp-001"
title: "Agent Runtime MVP 计划"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "runtime", "mvp"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# Agent Runtime MVP 计划

## 目标

跑通单 Agent 执行闭环：基于最小组织身份接任务、锁任务、组上下文、调用 C70、执行工具、写审计、完成状态流转。

## 阶段

| 阶段 | 组件 | 交付 |
|------|------|------|
| A0 最小身份上下文 | Company/Agent-lite | company_id、agent_id、role、budget、permission、status |
| A1 任务生命周期 | A10/A11/A02 | inbox、assigned、running、done、failed、suspended |
| A2 运行主循环 | A00/A01/A03/C70/X00 | context assemble、LLM stream、token event |
| A3 工具执行 | A20/A21/A22/A23/X04 | tool registry、policy、permission、guardrails |
| A4 质量门控 | A60/A63/X02 | Aegis review、loop detection、审计 |
| A5 审批恢复 | A62-lite/A10/X00 | HardLoop 与 KnowledgeReview 的最小工单与恢复链 |

## MVP 验收

- 一个任务只能被一个 Agent 锁定执行。
- 每个任务必须带 company_id、agent_id 和 role，支撑权限、预算和审计。
- AgentRuntime 不直接调用 ModelRouter/Provider，只通过 C70。
- 工具调用有权限判定和执行记录。
- HardLoop 可暂停任务并通过 A62-lite 创建审批工单。
- 完成任务后可触发 Aegis 审核。

## A62-lite 范围

P2 阶段只实现：

- `HardLoop`：异常暂停后的人工批准/拒绝与 `resume_task`。
- `KnowledgeReview`：AI 产出知识进入 Know 侧审核链的最小工单。

不在 P2 实现：

- 预算超限审批。
- 模型升级审批。
- 完整审批工作台、审批人分配、超时提醒。

这些能力归入 P4 的 `A62-full`。
