---
id: "spec-agent-task-scheduler-010"
title: "TaskScheduler 组件规范"
aliases: ["TaskScheduler规范", "任务调度器", "task-scheduler-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "task", "state-machine"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A10"
layer: "L1"
depends_on:
  - "A11"
optional_deps: []
conflicts_with: []
required_by:
  - "A02"
  - "A62"
  - "A70"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

<!-- @section: overview -->
# TaskScheduler 组件规范

## 1. 组件定位

`TaskScheduler` 管理 Agent 任务的生命周期：派遣、原子认领、状态流转、僵尸任务回收、HardLoop 强制挂起与审批恢复。

任务状态机为：`Inbox → Assigned → InProgress → QualityReview → Done`，异常分支为 `Failed` 与 `Suspended`。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
type TaskScheduler interface {
    FetchPending(ctx context.Context, agentID string, limit int) ([]Task, error)
    Claim(ctx context.Context, taskID, agentID string) (Task, LockReceipt, error)
    Transition(ctx context.Context, taskID string, from, to TaskStatus, reason string) error
    ForceInterrupt(ctx context.Context, taskID string, snapshot AgentContextSnapshot, reason string) error
    ResumeTask(ctx context.Context, taskID string, snapshot AgentContextSnapshot, humanGuidance string) error
    RequeueStale(ctx context.Context, now time.Time) (int, error)
}
```

<!-- @end-section -->

<!-- @section: state-machine -->
---

## 3. 状态机规则

| 转换 | 触发方 | 约束 |
|------|--------|------|
| `Inbox → Assigned` | 调度器 | 匹配 Agent 角色与能力 |
| `Assigned → InProgress` | AgentCoordinator | 必须先获取 TaskLock |
| `InProgress → QualityReview` | AgentCoordinator | `TaskResult=Completed` |
| `QualityReview → Done` | AegisReviewer | 仅 APPROVED 可进入 |
| `QualityReview → InProgress` | AegisReviewer | REJECTED 且未超过 max_cycles |
| `InProgress → Suspended` | `ForceInterrupt` | 仅 HardLoop / 审批门控可写 |
| `Suspended → InProgress` | `ResumeTask` | 必须有人类审批 |

<!-- @end-section -->

<!-- @section: invariants -->
---

## 4. 硬约束

- API 层不得直接写 `Done`，必须经 `QualityReview → Done`。
- `Suspended` 只能由 `ForceInterrupt` 写入。
- 僵尸任务最多回收 5 次，超过后转 `Failed`。
- 所有状态转换必须追加审计记录。

<!-- @end-section -->
