---
id: "spec-agent-task-lock-011"
title: "TaskLock 组件规范"
aliases: ["TaskLock规范", "任务锁", "task-lock-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "task", "lock", "cas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A11"
layer: "L1"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "A02"
  - "A10"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

<!-- @section: overview -->
# TaskLock 组件规范

## 1. 组件定位

`TaskLock` 提供任务执行的**原子租约锁**，防止多个 Agent 实例重复执行同一任务。锁记录使用 `owner_agent_id`、`locked_at`、`expires_at` 表示租约，默认锁时长 15 分钟。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
type TaskLock interface {
    TryAcquire(ctx context.Context, taskID, agentID string, ttl time.Duration) (LockReceipt, bool, error)
    Renew(ctx context.Context, receipt LockReceipt, ttl time.Duration) error
    Release(ctx context.Context, receipt LockReceipt) error
    ForceRelease(ctx context.Context, taskID, reason string) error
}

type LockReceipt struct {
    TaskID  string
    OwnerID string
    Token   string
    ExpiresAt time.Time
}
```

<!-- @end-section -->

<!-- @section: behavior -->
---

## 3. 行为契约

| 契约 | 说明 |
|------|------|
| CAS 获取 | 仅当 `status=assigned` 且锁为空或已过期时可获取 |
| Token 校验 | `Renew` / `Release` 必须校验锁 token，避免误释放 |
| 过期可回收 | `expires_at < now()` 的锁可被 `stale_task_requeue` 回收 |
| 幂等释放 | 重复释放同一 token 不报错 |
| 审计可查 | `ForceRelease` 必须写入原因和操作者 |

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[task-scheduler-spec|TaskScheduler 组件规范]]
- [[agent-coordinator-spec|AgentCoordinator 组件规范]]

<!-- @end-section -->
