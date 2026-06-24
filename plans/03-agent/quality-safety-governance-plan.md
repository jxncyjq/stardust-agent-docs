---
id: "plans-agent-quality-safety-governance-001"
title: "Agent 质量安全与治理计划"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "quality", "safety", "governance"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
related_docs:
  - path: "../../design/architecture/agent_components/aegis-reviewer-spec.md"
    relation: "implements"
  - path: "../../design/architecture/agent_components/approval-service-spec.md"
    relation: "implements"
  - path: "../../design/architecture/agent_components/trust-score-manager-spec.md"
    relation: "implements"
  - path: "../../design/architecture/agent_components/eval-engine-spec.md"
    relation: "implements"
---

# Agent 质量安全与治理计划

## 目标

把 A60-A64 拆成可逐步实现的质量、安全、审批和信任治理能力，避免 Agent 在具备工具执行能力后缺少边界。

## 组件分工

| 组件 | 权威职责 | 不负责 |
|------|----------|--------|
| A60 AegisReviewer | 对 Agent 输出和行动结果做质量评审 | 不直接修改任务状态，只返回审核结论 |
| A61 TrustScoreManager | 基于事件计算 Agent 信任分和派遣建议 | 不执行审批，不替代权限策略 |
| A62 ApprovalService | 创建审批工单、记录决策、恢复任务 | 不拥有 Know 知识状态机，不直接改知识状态 |
| A63 EvalEngine | 行为健康、循环、漂移和组件级 Eval | 不做人工审批 |
| A64 EvolutionEvaluator | 学习资产退化检测和能力评估 | 不直接固化 Gene/Capsule |

## 分期

### P2 A62-lite + A60/A63-lite

| 能力 | 验收 |
|------|------|
| HardLoop 检测 | A63-lite 检测循环后将任务置为 suspended |
| 质量评审 | A60 使用 C70 fake/recording 评估结果 |
| 最小审批 | A62-lite 创建 HardLoop/KnowledgeReview 工单 |
| 恢复链 | approved 后调用 TaskScheduler 恢复任务，denied 后任务失败 |
| 审计 | task/model/tool/review/approval 都能按 request_id 串联 |

### P4 A62-full + A61

| 能力 | 验收 |
|------|------|
| 七类门控 | HardLoop、KnowledgeReview、BudgetExceeded、ModelUpgrade、DangerousTool、WorkflowHumanGate、SkillInstall |
| 信任分 | 初始 0.7，事件驱动重算，支持冷却期 |
| 派遣建议 | allow、cautious、blocked 三档返回给 A10/A02 |
| 审批策略 | auto_allow 前缀、RBAC 角色覆盖、人工审批互不混淆 |
| 输出净化 | Aegis 结论和工具输出经 X05 净化后对外展示 |

### P5 A64

| 能力 | 验收 |
|------|------|
| 能力评估 | 5 维度评估 Agent 能力资产表现 |
| 退化检测 | 14 天窗口双条件触发 DegradationAlert |
| 进化门控 | 退化资产冻结，禁止被 A40 注入上下文 |

## 必测场景

| 场景 | 期望 |
|------|------|
| Aegis 拒绝任务结果 | Task 不进入 done，并写入 quality_review 事件 |
| HardLoop 触发 | Task suspended，ApprovalTicket open |
| 审批批准 | Task 从 suspended 恢复 running |
| 审批拒绝 | Task failed，并记录拒绝原因 |
| 低信任 Agent | TaskScheduler 返回 cautious/blocked 派遣建议 |
| 危险工具调用 | A21/A22/A62-full 共同阻断或转人工 |
