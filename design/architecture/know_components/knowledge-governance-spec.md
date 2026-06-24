---
id: "spec-know-knowledge-governance-050"
title: "KnowledgeGovernance 组件规范"
aliases: ["KnowledgeGovernance规范", "知识治理", "知识审核流程"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "governance", "review", "audit"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K50"
layer: "K5"
depends_on:
  - "K30"
  - "X02"
optional_deps:
  - "K51"
  - "K52"
  - "X00"
  - "A62"
conflicts_with: []
required_by: []
assembly_profiles:
  - standard
  - enterprise
---

# KnowledgeGovernance 组件规范

## 1. 组件定位

`KnowledgeGovernance` 管理 AI 与人类共同写入知识库时的版本、审核、冲突裁决和审计。所有 AI 产出知识默认不能直接进入正式库。

它是知识状态机的权威组件；若需要人工审批工作台、通知、超时提醒和审批人分配，则调用 `A62 ApprovalService` 创建 `KnowledgeReview` 工单。

## 2. 状态机

| 状态 | 语义 |
|------|------|
| PendingReview | AI 生成，等待人工审核 |
| UnderReview | 审核中 |
| Approved | 已批准，正式可检索 |
| Conflicting | 与已有知识冲突 |
| Watchlist | 冲突双版本均保留，等待实践验证 |
| Archived | 已归档 |
| Rejected | 已拒绝 |

## 3. 接口定义

```go
type KnowledgeGovernance interface {
    SubmitAIKnowledge(ctx context.Context, req SubmitKnowledgeRequest) (KnowledgeReviewTicket, error)
    Approve(ctx context.Context, ticketID string, decision ReviewDecision) error
    Archive(ctx context.Context, chunkID string, reason string) error
    Transition(ctx context.Context, chunkID string, from, to KnowledgeStatus, reason string) error
}
```

## 4. 流程

1. AI 提交知识 → `PendingReview`。
2. 调用 `ConflictDetector`。
3. 无冲突进入审核队列；有冲突转 `Conflicting`。
4. 需要人工审核时调用 `A62 ApprovalService` 创建 `KnowledgeReview` 工单。
5. A62 返回审批结论后，K50 执行知识状态 CAS 转换。
6. 状态转换写入 `ImmutableAuditLog`。
7. EventBus 广播治理事件。

## 5. 与 A62 ApprovalService 的分工

| 能力 | K50 KnowledgeGovernance | A62 ApprovalService |
|------|-------------------------|---------------------|
| 知识状态机 | 负责 | 不负责 |
| 冲突裁决规则 | 负责 | 不负责 |
| 创建审批工单 | 发起请求 | 负责创建和分配 |
| 审批通知、超时、工作台 | 不负责 | 负责 |
| 应用审批结果 | 负责 CAS 状态转换 | 只返回结论 |
| 审计 | 写入知识状态转换和裁决证据 | 记录审批事件并广播 |

## 6. 硬约束

- AI 产出不得直接写 `Approved`。
- `Conflicting → Approved` 必须带裁决记录。
- `Archived` 不参与默认检索，但可在审计视图查询。
- 状态转换必须 CAS，避免并发审核覆盖。
