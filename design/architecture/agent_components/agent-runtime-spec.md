---
id: "spec-agent-runtime-001"
title: "AgentRuntime 组件规范"
aliases: ["AgentRuntime规范", "Agent主循环", "agent-runtime-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "llm-loop", "tool-execution", "legion"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A01"
layer: "L0"
depends_on:
  - "A00"   # CognitiveCore — 上下文组装（持有所有权）
  - "A20"   # ToolRegistry — 工具执行
  - "C70"   # MaasInferenceClient — MaaS 稳定推理端口
  - "X00"   # EventBus — 流式 token 推送与学习事件发布
optional_deps:
  - "A63"   # EvalEngine — 缺失时不执行收敛比检测，永不触发 SoftLoop/HardLoop
  - "A22"   # PermissionEnforcer — 缺失时跳过权限批量校验（仅开发环境可用）
  - "A23"   # ToolGuardrails — 缺失时不执行 before/after 调用钩子
  - "A40"   # MemoryProvider — 缺失时不写入情景记忆
conflicts_with: []
required_by:
  - "A02"   # AgentCoordinator 通过 Mutex 持有并调用 AgentRuntime
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

<!-- @section: overview -->
# AgentRuntime 组件规范

## 1. 组件定位

`AgentRuntime` 是 **单次任务执行引擎**，负责 LLM 调用循环本身。它是 Agent 引擎中最核心的组件，也是职责最单一的组件：

**负责**：LLM 调用循环、工具执行链、上下文管理、流式 token 推送、学习事件发布  
**不负责**：任务来源管理、原子锁、任务状态流转、审计链写入、路由决策（这些均由 `AgentCoordinator` 负责）

```
AgentCoordinator（任务编排层，A02）
    │  传入：Task（已锁定）+ RoutingDecision（已选定模型）
    │  接收：TaskResult（Completed / AnomalyDetected / BudgetExhausted / Interrupted）
    ▼
AgentRuntime（执行层，A01）
    │
    ├── CognitiveCore(A00)         → 上下文组装（首次）和循环检查点（SoftLoop 时）
    ├── ToolRegistry(A20)          → 工具路由与执行
    ├── PermissionEnforcer(A22)    → 批量权限检查（可选）
    ├── ToolGuardrails(A23)        → 执行前/后钩子（可选）
    ├── EvalEngine(A63)            → 实时收敛比检测（可选）
    ├── MaasInferenceClient(C70)   → MaaS 稳定推理端口，不直接调用 ModelRouter/Provider
    └── EventBus(X00)              → 流式 token 推送 + LearningEvent 发布
```

**读者**：实现 Agent 主循环的工程师、需要扩展工具执行链的开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// AgentRuntime 是单次任务执行引擎。
// 调用方（AgentCoordinator）持有 Mutex<AgentRuntime>，通过 Mutex 获取 &mut 后调用 RunTask。
// RunTask 不是并发安全的（ContextCompressor.cycle_count 需要跨迭代持久化）。
type AgentRuntime interface {
    // RunTask 执行一次完整的 LLM 调用循环，直到任务完成、超预算、被中断或检测到异常。
    // task 已由 AgentCoordinator 原子锁定；routing 是已完成的路由决策（不再重新路由）。
    // 返回 TaskResult，AgentCoordinator 依据结果分支处理后续状态流转。
    // 需要 &mut self（ContextCompressor.cycle_count 跨迭代持久化）。
    RunTask(ctx context.Context, task *Task, routing *RoutingDecision) (TaskResult, error)

    // Interrupt 发送中断信号，RunTask 在下一次迭代检查时返回 TaskResult_Interrupted。
    // 并发安全：通过 AtomicBool 实现，可从外部 goroutine 调用。
    Interrupt()
}
```

<!-- @end-section -->

<!-- @section: data-types -->
---

## 3. 数据类型定义

### 3.1 路由决策（输入）

```go
// RoutingDecision 是 AgentCoordinator 在 Step 4 通过 IntelligentRouter 完成的路由决策。
// 传入 RunTask 后不再变更——AgentRuntime 将路由意图交给 C70，不直接调用 ModelRouter/Provider。
type RoutingDecision struct {
    SelectedModel  string          // 提供商侧实际模型 ID
    ProviderID     string          // 提供商 ID
    Tier           ModelTier       // 路由决定的 Tier（Light/Standard/Advanced/Flagship）
    RoutingTrace   *RoutingTrace   // 路由决策的 10 步中间状态（可观测）
}
```

### 3.2 任务结果（输出）

```go
// TaskResult 是 RunTask() 的返回值，驱动 AgentCoordinator 的后续分支处理。
type TaskResult int

const (
    // TaskResult_Completed Agent 完成任务，产出已准备好进入 Aegis 质量审核。
    TaskResult_Completed TaskResult = iota

    // TaskResult_AnomalyDetected 检测到 HardLoop（收敛比 > 10），强制中断。
    // AgentCoordinator 收到后触发 §6.2.1 HardLoop 响应链（写 Suspended，创建审批工单）。
    TaskResult_AnomalyDetected

    // TaskResult_BudgetExhausted 迭代次数超出 max_iterations。
    // 轻量 LearningEvent 已在内循环 Step 1 提前发布。
    TaskResult_BudgetExhausted

    // TaskResult_Interrupted 收到外部中断信号（用户取消 / force_interrupt）。
    // 轻量 LearningEvent 已在内循环 Step 2 提前发布。
    TaskResult_Interrupted
)

// TaskResultDetail 包含 TaskResult 及附加上下文。
type TaskResultDetail struct {
    Result      TaskResult
    Output      string          // 任务产出（Completed 时有值）
    Reason      string          // 终止原因（非 Completed 时有值）
    LoopRatio   float64         // HardLoop 时的收敛比值
    Iterations  int             // 实际执行的迭代轮数
}
```

### 3.3 学习事件（发布到 EventBus）

```go
// LearningEvent 是 AgentRuntime 发布到 EventBus 的学习信号。
// gep_failure_scan 后台任务（每 15 分钟）消费此队列触发 GEP 周期。
type LearningEvent struct {
    AgentID     string
    TaskID      string
    Signal      LearningSignal    // 学习信号枚举
    Reason      FailureReason     // 失败原因（仅 Failure 信号时有值）
    EpisodicRef string            // 对应的情景记忆 ID（完整记录时有值）
    IsLightweight bool            // true=轻量版（不含完整轨迹），false=完整版
    PublishedAt time.Time
}

type LearningSignal int
const (
    LearningSignal_Success       LearningSignal = iota // 任务成功完成
    LearningSignal_Failure                              // 任务失败（含 BudgetExhausted/Interrupted）
    LearningSignal_HardLoopFailure                      // 硬循环失败（由 HardLoop 响应链发布）
)

type FailureReason int
const (
    FailureReason_BudgetExhausted FailureReason = iota
    FailureReason_Interrupted
    FailureReason_HardLoop
    FailureReason_ToolError
)
```

<!-- @end-section -->

<!-- @section: run-task-loop -->
---

## 4. RunTask 执行循环规范

`RunTask` 是本组件的核心，包含**迭代循环**和**7 步防护机制**。

### 4.1 循环前初始化（只执行一次）

```
1. 调用 cognitive_core.AssembleContext(req) → 获取 AssembledContext
   （系统提示、初始消息列表、可用工具规格）
2. 初始化 message_history = AssembledContext.InitMessages
   （循环外初始化，后续迭代只追加，不重建）
```

### 4.2 每次迭代的 7 步防护

```
─────────────────────────────────────────
 迭代 N（N = 1, 2, 3, ...）
─────────────────────────────────────────

Step 1. 迭代预算检查
  if N > task.max_iterations:
    发布轻量 LearningEvent(Signal=Failure, Reason=BudgetExhausted, IsLightweight=true)
    返回 TaskResult_BudgetExhausted
    （不写 EpisodicMemory——避免记录不完整状态）

Step 2. 中断检查
  if interrupt_flag.load(Acquire) == true:
    发布轻量 LearningEvent(Signal=Failure, Reason=Interrupted, IsLightweight=true)
    返回 TaskResult_Interrupted

Step 3. 流式 LLM 调用
  使用 routing.selected_model（已由 AgentCoordinator 确定，不再重新路由）
  inference_client.Stream(PurposeAgentLoop, routing, message_history)
    → 每个 token delta 推送到 event_bus（"stream.token" 事件，UI 实时展示）
    → 收集完整 LLM 响应（包含 tool_calls、stop_reason）

Step 4. 工具执行链（当 LLM 响应含 tool_calls 时执行）
  对每个 tool_call 并发执行：
    a. tool_guardrails.before_call(tool_call)         ← 可选，看门狗前置检查
    b. permission_enforcer.check_batch([tool_call])   ← 可选，批量权限校验
    c. check_approval(tool_call, ExecutionPolicy)     ← 按 ApprovalPolicy 决定是否需要用户批准
    d. tool_registry.execute(tool_call)               ← 实际执行
    e. tool_guardrails.after_call(tool_call, result)  ← 可选，看门狗后置钩子
  将所有工具结果追加到 message_history

Step 5. 收敛比实时检测（每次工具执行后立即检测）
  eval_engine.eval_trace(task_id, soft_loop_reset_at) → LoopingStatus
    过滤：仅统计 soft_loop_reset_at（或 loop_reset_at）之后的 mcp_call_log 记录

  match LoopingStatus:
    Normal(ratio ≤ 5.0)  → 继续
    SoftLoop(5.0 < ratio ≤ 10.0) →
      写入 tasks.soft_loop_reset_at = now()
      cognitive_core.ForceCheckpoint(message_history) → 循环检查点
      继续（不中断循环）
    HardLoop(ratio > 10.0) →
      返回 TaskResult_AnomalyDetected
      （LearningEvent 由 AgentCoordinator 的 HardLoop 响应链发布）

Step 6. 工作记忆更新（可选，非关键路径）
  working_memory.append(本次迭代的工具调用摘要)

Step 7. 终止检测
  if LLM 响应的 stop_reason == STOP（无工具调用）:
    post_task_learning():
      a. 写入完整情景记忆到 EpisodicMemoryStore
         （task_description + tool_call_sequence + result）
      b. 发布完整 LearningEvent(Signal=Success/Failure, IsLightweight=false)
    返回 TaskResult_Completed

  else（还有工具调用结果需要 LLM 继续处理）:
    继续下一次迭代
```

### 4.3 执行上下文的跨迭代积累

- `message_history` 在**循环外**初始化，每次迭代只**追加**（LLM 回复 + 工具结果）
- LLM 在每轮迭代都能看到完整的前序工具调用链，支持多轮推理
- `ContextCompressor`（通过 `CognitiveCore`）在 80% 窗口利用率时自动压缩，或在 SoftLoop 时强制检查点

<!-- @end-section -->

<!-- @section: tool-execution-chain -->
---

## 5. 工具执行链细节

### 5.1 check_approval 逻辑

```go
// check_approval 根据 ExecutionPolicy 决定是否需要等待用户批准。
func check_approval(tool_call *ToolCall, policy *ExecutionPolicy) ApprovalDecision {
    // 1. 检查 auto_allow_prefixes
    if policy.IsAutoAllowed(tool_call.Name) {
        return ApprovalDecision_Auto
    }
    // 2. 按 ApprovalPolicy 决策
    switch policy.ApprovalPolicy {
    case AutoAllow:
        return ApprovalDecision_Auto
    case OnRequest:
        if tool_call.IsHighRisk() {
            return ApprovalDecision_RequireApproval
        }
        return ApprovalDecision_Auto
    case Untrusted:
        return ApprovalDecision_RequireApproval  // 每次都要批准
    case Never:
        return ApprovalDecision_Blocked          // 拒绝所有工具
    }
}
```

### 5.2 SandboxMode 执行隔离

| SandboxMode | 工具执行方式 |
|-------------|------------|
| `ReadOnly` | 工具只能读取，写操作由 PermissionEnforcer 拦截 |
| `WorkspaceWrite` | 只能修改工作区目录内文件，其他路径拒绝 |
| `DangerFullAccess` | 所有操作允许（含系统命令），记录所有执行到审计日志 |
| `ExternalSandbox` | 在隔离容器中执行（委托给 CliAgentAdapter 的 `spawn_sandboxed`） |

### 5.3 并发工具执行

- 同一 LLM 响应中的多个 tool_calls 支持并发执行（`tokio::join!` / Go `errgroup`）
- 并发执行时 PermissionEnforcer.check_batch 接收整批 tool_calls，一次性批量校验
- 若任意工具执行失败：
  - Critical 错误（越权、沙盒违规）→ 中止全部并发工具，返回错误
  - Non-critical 错误（工具本身返回错误）→ 收集所有结果（成功+失败），统一追加到 message_history

<!-- @end-section -->

<!-- @section: errors -->
---

## 6. 错误定义

```go
var (
    // ErrLLMCallFailed LLM 调用失败（网络错误、提供商错误）。
    // AgentRuntime 内部处理重试（委托给 MaasInferenceClient），超出重试后返回此错误。
    ErrLLMCallFailed = errors.New("agent_runtime: llm call failed")

    // ErrToolExecutionFailed 工具执行发生 Critical 级别错误，任务终止。
    ErrToolExecutionFailed = errors.New("agent_runtime: tool execution failed (critical)")

    // ErrPermissionDenied 工具调用被 PermissionEnforcer 拦截。
    ErrPermissionDenied = errors.New("agent_runtime: tool call permission denied")
)
```

<!-- @end-section -->

<!-- @section: observability -->
---

## 7. 可观测性

| 事件 | 发布到 EventBus | 说明 |
|------|----------------|------|
| 每个 token delta | `stream.token` | 流式 UI 展示 |
| 工具调用开始 | `tool.call.start` | 工具名、参数摘要 |
| 工具调用完成 | `tool.call.done` | 工具名、耗时、成功/失败 |
| SoftLoop 触发 | `task.soft_loop` | ratio 值、checkpoint 摘要 |
| HardLoop 触发 | `task.hard_loop` | ratio 值（AgentCoordinator 响应链使用） |
| 任务完成 | `task.completed` | 产出摘要 |
| LearningEvent 发布 | `agent.learning` | 信号类型、轻量/完整标记 |

**写入 mcp_call_log 表**（每次工具调用）：
- `agent_name`, `tool_name`, `input`, `output`, `success`, `error`, `duration_ms`
- EvalEngine 的收敛比检测从此表读取数据（仅统计 `soft_loop_reset_at` 之后的记录）

<!-- @end-section -->
