---
id: "spec-agent-approval-service-062"
title: "ApprovalService 组件规范"
aliases: ["ApprovalService规范", "审批服务", "approval-service-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "approval", "governance"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A62"
layer: "L6"
depends_on:
  - "A10"
  - "X00"
optional_deps: []
conflicts_with: []
required_by:
  - "A02"
  - "A70"
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# ApprovalService 组件规范

## 1. 组件定位

`ApprovalService` 管理平台统一的人类审批工单，尤其是 HardLoop 触发的 `AnomalyEscalation` 恢复链。它负责“谁来审、审到什么状态、如何通知和恢复任务”，不负责各业务对象自身的状态机。

<!-- @end-section -->

<!-- @section: gate-types -->
---

## 2. 七大审批门控

| 类型 | 场景 |
|------|------|
| Recruitment | Agent 请求创建下属 |
| BudgetOverrun | 预计成本超阈值 |
| StrategyReview | 管理层方案产出 |
| ModelUpgrade | 请求更高等级模型 |
| AnomalyEscalation | HardLoop / 连续异常 |
| QualityGate | 交付物质量争议 |
| KnowledgeReview | 知识库写入/冲突裁决需要人工审核 |

> `KnowledgeReview` 只表示审批工单类型。知识 chunk 的 `PendingReview / Conflicting / Approved / Rejected` 等状态由 `K50 KnowledgeGovernance` 管理。

<!-- @end-section -->

<!-- @section: contracts -->
---

## 3. 行为契约

- 不提供自动批准接口，审批必须有人类主体。
- Approved HardLoop 任务通过 `TaskScheduler.ResumeTask` 恢复。
- Denied 任务转 `Failed` 并广播。
- Allowlist 更新使用 CAS，避免并发覆盖。
- 对 `KnowledgeReview`，A62 只返回审批结论和审批人信息，不直接修改知识库。

## 4. 与 K50 KnowledgeGovernance 的分工

| 能力 | A62 ApprovalService | K50 KnowledgeGovernance |
|------|---------------------|-------------------------|
| 创建人工审批工单 | 负责 | 调用 A62 创建 |
| 分配审批人、通知、超时提醒 | 负责 | 不负责 |
| 记录审批结论 | 负责 | 读取结论作为状态转换输入 |
| 知识状态转换 | 不负责 | 负责，使用 CAS 更新 |
| 冲突裁决规则 | 不负责 | 负责 |
| 审计证据链 | 记录审批事件并广播 | 写入知识状态转换与裁决证据到 X02 |

<!-- @end-section -->
