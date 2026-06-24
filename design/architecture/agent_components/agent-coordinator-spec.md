---
id: "spec-agent-coordinator-002"
title: "AgentCoordinator 组件规范"
aliases: ["AgentCoordinator规范", "任务编排协调层", "agent-coordinator-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "heartbeat", "task-lifecycle", "legion"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A02"
layer: "L0"
depends_on:
  - "A01"   # AgentRuntime — 通过 Mutex 持有并调用
  - "A10"   # TaskScheduler — 任务获取、状态流转、resume
  - "A11"   # TaskLock — 原子锁定/释放
  - "X02"   # ImmutableAuditLog — 每次心跳写入不可篡改审计链
optional_deps:
  - "A61"   # TrustScoreManager — 缺失时跳过 Step 5 信任约束检查
  - "A62"   # ApprovalService — 缺失时 HardLoop 不创建审批工单（仅标记 Suspended）
conflicts_with: []
required_by: []   # 顶层编排组件，不被其他组件依赖
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

<!-- @section: overview -->
# AgentCoordinator 组件规范

## 1. 组件定位

`AgentCoordinator` 是 **任务编排协调层**，负责任务生命周期的"外层"管理：心跳协议、原子锁定、路由决策、状态流转、审计链写入。它**不包含 LLM 调用循环**（由 `AgentRuntime` 负责）。

两者职责对比：

| 关注点 | AgentCoordinator（本组件）| AgentRuntime（A01）|
|--------|--------------------------|-------------------|
| 任务来源 | ✅ fetch_pending | ❌ |
| 原子锁定 | ✅ try_acquire | ❌ |
| 路由决策 | ✅ IntelligentRouter.route() | ❌（只使用决策结果）|
| 信任约束 | ✅ apply_trust_constraints | ❌ |
| LLM 调用循环 | ❌ | ✅ run_task |
| 工具执行 | ❌ | ✅ |
| 任务状态写入 | ✅ in_progress → quality_review 等 | ❌ |
| 审计链写入 | ✅ 每次心跳写入 | ❌ |
| HardLoop 响应链 | ✅ 触发 force_interrupt + 工单 | ❌（只返回 AnomalyDetected）|

```
外部（心跳触发器 / 调度器）
    │
    ▼
AgentCoordinator.ExecuteHeartbeat(agent)
    │
    ├── TaskScheduler(A10)    → fetch_pending / 状态流转 / resume_task
    ├── TaskLock(A11)         → try_acquire / release
    ├── IntelligentRouter     → route（纯决策，不调用 LLM）
    ├── TrustScoreManager(A61)→ apply_trust_constraints（可选）
    ├── AgentRuntime(A01)     → run_task（通过 Mutex 串行调用）
    ├── ApprovalService(A62)  → create_ticket（HardLoop 时，可选）
    └── ImmutableAuditLog(X02)→ append（所有分支均写入）
```

**读者**：理解 Agent 任务全链路的架构师、实现心跳协议的工程师。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// AgentCoordinator 是任务编排协调层。
// 并发安全：可并发调用（多个 Agent 实例各自的心跳并发），
// 但同一 AgentCoordinator 实例内部通过 Mutex<AgentRuntime> 保证串行执行。
type AgentCoordinator interface {
    // ExecuteHeartbeat 执行一次 Agent 心跳，完成完整的九步任务生命周期。
    // agent 是当前执行的 Agent 实体。
    // 返回 HeartbeatResult，供调度器判断是否继续派发任务。
    ExecuteHeartbeat(ctx context.Context, agent *Agent) (*HeartbeatResult, error)

    // ResumeTask 在 ApprovalService 批准后，从 Suspended 状态恢复任务。
    // 由 ApprovalService.respond_approve() 内部调用（非外部直接调用）。
    ResumeTask(ctx context.Context, taskID string, snapshot *AgentContextSnapshot) error
}
```

<!-- @end-section -->

<!-- @section: data-types -->
---

## 3. 数据类型定义

```go
// HeartbeatResult 描述本次心跳的执行结果。
type HeartbeatResult struct {
    Status      HeartbeatStatus
    TaskID      string          // 处理的任务 ID（NoTask 时为空）
    TaskResult  TaskResult      // AgentRuntime 的返回值
    TrustScore  float64         // 本次心跳时的有效信任分（可观测）
    AuditEntryID string         // 写入审计链的条目 ID
}

type HeartbeatStatus int
const (
    HeartbeatStatus_Completed       HeartbeatStatus = iota // 任务完成，进入质量审核
    HeartbeatStatus_NoTask                                  // 无待处理任务，心跳空转
    HeartbeatStatus_TrustBlocked                           // 信任分 < 0.3，跳过执行
    HeartbeatStatus_LockContended                          // 任务已被其他实例锁定
    HeartbeatStatus_Suspended                              // HardLoop，任务挂起
    HeartbeatStatus_Failed                                 // 任务执行失败
)

// AgentContextSnapshot 是 HardLoop 挂起时序列化的执行上下文快照，用于恢复。
type AgentContextSnapshot struct {
    MessageHistory  []Message     // 压缩版消息历史
    WorkingMemory   string        // 工作记忆当前内容
    IterationCount  int           // 已执行的迭代轮数
    SnapshotAt      time.Time
}
```

<!-- @end-section -->

<!-- @section: heartbeat-nine-steps -->
---

## 4. ExecuteHeartbeat 九步流程

```
─────────────────────────────────────────────────────────────────
 ExecuteHeartbeat(ctx, agent)
─────────────────────────────────────────────────────────────────

Step 1. 身份确认
  verify_identity(agent)
  → 检查 agent.ID 在 agents 表中存在且处于 active 状态
  → 失败时返回 ErrAgentNotFound，不执行后续步骤

Step 2. 获取待办任务
  task = task_scheduler.fetch_pending(agent.id)
  → 查询 status='assigned' AND assigned_to=agent.id，取优先级最高的一条
  → 无任务时返回 HeartbeatStatus_NoTask（正常，等待下次心跳）

Step 3. 原子锁定任务
  lock = task_lock.try_acquire(task.id, agent.id, max_task_lock_duration=15min)
  → 使用原子 RETURNING UPDATE（CAS）防竞态：
    UPDATE tasks SET status='in_progress', locked_by=agent.id
    WHERE id=task.id AND status='assigned'
    RETURNING *
  → 返回 0 行（其他实例已锁定）→ 返回 HeartbeatStatus_LockContended

Step 4. 路由决策
  routing = intelligent_router.route(task, agent.role)
  → 完整 10 步路由流水线（见 agent-engine-design-nocode.md §1.8.5）
  → 纯决策，不调用 LLM，不消耗配额
  → 失败（无可用提供商）→ 释放锁，任务回退到 Assigned，返回错误

Step 5. 信任约束检查（可选，依赖 TrustScoreManager）
  score = trust_score_manager.effective_score(agent.id)
  if score < 0.3:
    task_lock.release(lock)   ← 释放锁，任务保持 Assigned
    return HeartbeatStatus_TrustBlocked
    ⚠️  注意：TrustBlocked 不将任务转为 Failed，任务停留在 Assigned
    若 Agent 长期 score < 0.3，需通过 TrustScoreManager 的 assign_timeout 超时机制清理积压

  if 0.3 ≤ score < 0.5:
    routing = routing.downgrade_to_light()  ← 强制降为 Light Tier
    task = task.restrict_to_low_priority()  ← 任务优先级约束
    申请全工具审批（设置 ExecutionPolicy.approval_policy = Untrusted）

Step 6. 执行工作
  result_detail = agent_runtime.lock().run_task(ctx, task, routing)
  → 通过 Mutex 获取 &mut AgentRuntime（串行执行，保证 cycle_count 一致性）
  → AgentRuntime 执行完整的 LLM 调用循环，返回 TaskResultDetail

Step 7. 按执行结果分支处理任务状态

  match result_detail.Result:

  [Completed]
    task_scheduler.update_status(task.id, InProgress → QualityReview)
    → 写入 quality_reviews 队列（等待 Aegis 审核，§2.10.2）
    → return HeartbeatStatus_Completed

  [AnomalyDetected (HardLoop)]
    ── HardLoop 响应链（§6.2.1）──
    a. task_scheduler.force_interrupt(task.id):
         tasks.status = Suspended（原子 CAS）
         tasks.suspended_at = now()
         tasks.suspend_reason = "HardLoop: ratio={ratio}"
         tasks.suspend_snapshot = serialize(agent_context)
    b. 确认 AgentRuntime 已退出循环（已通过返回 TaskResult_AnomalyDetected 退出）
       若有并发子任务：cancellation_token.cancel()，最多等待 tool_timeout_secs
    c. approval_service.create_ticket(AnomalyEscalation, task.id, evidence)（可选）
    d. event_bus.publish("task.suspended", {task_id, reason})  ← SSE 广播管理员
    e. event_bus.publish(LearningEvent{Signal=HardLoopFailure}) ← GEP 学习信号
    → return HeartbeatStatus_Suspended

  [BudgetExhausted | Interrupted]
    task_scheduler.update_status(task.id, InProgress → Failed)
    → 写入 task.failure_reason
    → LearningEvent 已在 AgentRuntime Step 1/2 提前发布，无需重复发布
    → return HeartbeatStatus_Failed

Step 8. 写入审计链（所有分支均执行）
  immutable_audit_log.append(AuditEntry{
    agent_id: agent.id,
    task_id:  task.id,
    result:   result_detail.Result,
    routing:  routing,
    duration: elapsed,
    trust_score: score,
  })

Step 9. 释放任务锁（所有分支均执行）
  task_lock.release(lock)
  → DELETE FROM task_locks WHERE task_id = ? AND agent_id = ?
```

### 4.1 关键约束

| 约束 | 实现方式 |
|------|---------|
| Step 5 TrustBlocked 时锁释放 | try_acquire 之后到 run_task 之前的 trust check，失败即释放 |
| Step 7 AnomalyDetected 只由此处写 Suspended | `tasks.status = Suspended` 仅在 `force_interrupt()` 内执行，API 层 403 |
| Step 8 审计链必须写入 | 即使 Step 9 释放锁失败，审计条目也应已写入（Step 8 先于 Step 9）|
| Step 9 锁释放幂等 | 若 task_lock.release 失败（锁已超时自然过期），不阻塞流程 |

<!-- @end-section -->

<!-- @section: resume-task -->
---

## 5. ResumeTask 流程（HardLoop 批准后）

```
ApprovalService.respond_approve(task_id)
  → 调用 AgentCoordinator.ResumeTask(task_id, snapshot)
     a. 从 tasks.suspend_snapshot 反序列化 AgentContextSnapshot
     b. 可选注入人类指导（降低循环风险的补充指令）
     c. 重置收敛比计数器：写入 tasks.loop_reset_at = now()
        （Layer 2 收敛比查询 mcp_call_log 时自动过滤此时间戳之前的记录）
        宽限期：loop_reset_at 后 5 分钟内 ratio 恒为 0
     d. 原子 CAS：tasks.status = Suspended → InProgress
     e. 将任务重新放入 fetch_pending 队列，下次心跳继续执行
```

<!-- @end-section -->

<!-- @section: errors -->
---

## 6. 错误定义

```go
var (
    ErrAgentNotFound      = errors.New("agent_coordinator: agent not found or inactive")
    ErrRoutingFailed      = errors.New("agent_coordinator: no available model for routing")
    ErrLockContended      = errors.New("agent_coordinator: task lock already acquired by another instance")
    ErrInvalidTaskState   = errors.New("agent_coordinator: unexpected task state transition")
    ErrResumeNotSuspended = errors.New("agent_coordinator: cannot resume task not in Suspended state")
)
```

<!-- @end-section -->

<!-- @section: observability -->
---

## 7. 可观测性

**审计链条目（写入 ImmutableAuditLog）**：
- `agent_id`, `task_id`, `step` (heartbeat_step), `action`, `result`, `duration_ms`, `routing_trace`

**EventBus 事件**：

| 事件名 | 触发时机 | 数据 |
|--------|---------|------|
| `task.in_progress` | Step 3 锁定成功 | task_id, agent_id |
| `task.quality_review` | Step 7 Completed | task_id |
| `task.suspended` | Step 7 AnomalyDetected | task_id, reason, ratio |
| `task.failed` | Step 7 BudgetExhausted/Interrupted | task_id, reason |
| `task.trust_blocked` | Step 5 TrustBlocked | agent_id, score |

<!-- @end-section -->
