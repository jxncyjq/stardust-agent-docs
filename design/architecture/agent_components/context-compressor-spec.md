---
id: "spec-agent-context-compressor-003"
title: "ContextCompressor 组件规范"
aliases: ["ContextCompressor规范", "上下文压缩器", "context-compressor-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "context", "compression", "legion"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A03"
layer: "L0"
depends_on: []
optional_deps:
  - "C70"               # MaasInferenceClient — 用于辅助摘要和循环检查点；缺失时只使用零 LLM 压缩策略
conflicts_with: []
required_by:
  - "A00"               # CognitiveCore 持有 ContextCompressor 所有权，通过 ForceCheckpoint 委托
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

<!-- @section: overview -->
# ContextCompressor 组件规范

## 1. 组件定位

`ContextCompressor` 负责 Agent 执行上下文的**压缩管理**：当消息历史超出模型上下文窗口阈值时，按照四层策略逐级压缩，保证 LLM 调用始终在有效窗口内进行。

```
AgentRuntime（每次迭代追加消息）
    │
    ▼ 超过 80% 窗口利用率 / SoftLoop 触发
CognitiveCore.ForceCheckpoint()
    │
    ▼ 委托
ContextCompressor
    ├── 第 1 层：TrimOldToolOutputs  （零 LLM 调用）
    ├── 第 2 层：ProtectHeadTail     （零 LLM 调用）
    ├── 第 3 层：AuxiliarySummary    （通过 C70 的轻量 LLM 调用）
    └── 第 4 层：CycleCheckpoint     （通过 C70 的轻量 LLM 调用，强制时直接执行）
```

**与 CognitiveCore 的关系**：
- `ContextCompressor` 持有权在 `CognitiveCore` 内
- `CognitiveCore.ForceCheckpoint()` 直接委托给 `ContextCompressor.force_checkpoint()`
- `ContextCompressor` 的 `cycle_count` 需要跨迭代持久化，这是 `AgentRuntime.RunTask` 需要 `&mut self` 的根本原因

**读者**：实现上下文管理策略的工程师、需要调整压缩行为的架构师。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// ContextCompressor 管理 Agent 执行上下文的压缩生命周期。
// 通过 CognitiveCore 持有，不独立注册到 ComponentRegistry。
// 非并发安全：由 CognitiveCore 顺序调用（AgentRuntime 串行执行保证）。
type ContextCompressor interface {
    // ShouldCompress 检测是否需要压缩（80% 窗口利用率阈值）。
    // 由 AgentRuntime 在每次迭代后调用，决定是否触发 Compress。
    ShouldCompress(history *MessageHistory, maxContextTokens int) bool

    // Compress 按当前策略压缩消息历史。
    // 始终先执行第 1 层（零成本裁剪）；若仍超阈值，按 strategy 选择后续层级。
    // 返回压缩后的消息历史和压缩报告。
    Compress(ctx context.Context, history *MessageHistory) (*MessageHistory, *CompressionReport, error)

    // ForceCheckpoint 强制执行第 4 层（CycleCheckpoint），忽略 80% 阈值。
    // 由 CognitiveCore 在 SoftLoop 触发时委托调用。
    ForceCheckpoint(ctx context.Context, history *MessageHistory) (*MessageHistory, *CheckpointResult, error)

    // CycleCount 返回当前循环计数（已经历的检查点次数）。
    CycleCount() int

    // SetStrategy 动态调整压缩策略（由 AgentDef 配置驱动）。
    SetStrategy(strategy CompressionStrategy)
}
```

<!-- @end-section -->

<!-- @section: data-types -->
---

## 3. 数据类型定义

### 3.1 压缩策略

```go
// CompressionStrategy 决定在第 1 层之后使用哪种压缩路径。
type CompressionStrategy int

const (
    // StrategyTrimAndProtect 第 1 层 + 第 2 层（无 LLM 调用，适合低成本场景）。
    StrategyTrimAndProtect CompressionStrategy = iota

    // StrategyAuxiliarySummary 第 1 层 + 第 2 层 + 第 3 层（轻量 LLM 摘要）。
    StrategyAuxiliarySummary

    // StrategyCycleCheckpoint 第 1 层 + 第 4 层（深度检查点，替代全部历史）。
    // 适合长任务（>25 轮迭代）。
    StrategyCycleCheckpoint
)
```

### 3.2 压缩报告

```go
// CompressionReport 记录一次压缩操作的详情，写入审计日志。
type CompressionReport struct {
    Strategy      CompressionStrategy
    LayersApplied []int           // 实际执行的层次（如 [1, 2, 4]）
    TokensBefore  int
    TokensAfter   int
    Summary       string          // 摘要内容（第 3/4 层时有值）
    CycleBriefing *CycleBriefing  // 检查点摘要（第 4 层时有值）
    CompressedAt  time.Time
}

// CycleBriefing 是循环检查点生成的周期摘要。
// 保留足够上下文，让下一循环了解前序工作。
type CycleBriefing struct {
    CycleIndex    int
    ObjectiveDone string   // 本周期完成了什么
    KeyFindings   []string // 关键发现（工具调用结果摘要）
    NextFocus     string   // 下一步方向
    TokensSaved   int
}
```

<!-- @end-section -->

<!-- @section: four-layer-strategy -->
---

## 4. 四层压缩策略规范

### 第 1 层：TrimOldToolOutputs（零 LLM 调用）

**触发**：始终先执行  
**机制**：裁剪旧工具输出，只保留最新 N 条（N 由配置决定，默认保留最近 5 条工具结果）

```
规则：
  - 保留所有 user / assistant 消息
  - 识别 tool_result 消息，按时间倒序保留最新 5 条
  - 将超出范围的 tool_result 内容替换为 "[截断：工具结果已压缩]"
  - 对应的 assistant 消息中的 tool_use 引用保持不变（只截断结果，不截断调用）
```

### 第 2 层：ProtectHeadTail（零 LLM 调用）

**触发**：第 1 层压缩后仍超阈值  
**机制**：保护首尾消息，截断中间部分

```
规则：
  - 保护 HEAD：保留前 3 条消息（任务初始化上下文）
  - 保护 TAIL：保留后 10 条消息（近期执行上下文）
  - 中间消息替换为单条摘要占位符：
    "[省略 N 条消息（约 M tokens）]"
  - 不调用 LLM，纯文本替换
```

### 第 3 层：AuxiliarySummary（低 LLM 调用成本）

**触发**：第 2 层压缩后仍超阈值  
**机制**：用辅助模型（轻量级）对中间历史生成摘要

```
规则：
  - 摘要目标：提取工具调用结论、已完成的子任务、关键发现
  - 摘要模型：通过 `C70 MaasInferenceClient` 请求 Light Tier（成本最低）
  - 摘要 token 上限：800 tokens（约为被替换历史的 10%）
  - 摘要插入位置：HEAD 之后、TAIL 之前
  - 原中间消息丢弃（不可恢复，但摘要存入 CompressionReport.Summary 持久化）
```

### 第 4 层：CycleCheckpoint（低 LLM 调用成本）

**触发**：
- `ShouldCompress` 返回 true 且当前 strategy = StrategyCycleCheckpoint（cycle_count 到达 cycle_length）
- 或 `ForceCheckpoint()` 被调用（SoftLoop 触发，忽略阈值）

**机制**：用 CycleBriefing 替代全部历史，重置计数

```
执行流程：
  Step 1. 通过 `C70 MaasInferenceClient.Complete(PurposeContextSummary)` 生成 CycleBriefing：
    - ObjectiveDone（本周期完成了什么）
    - KeyFindings（工具调用的关键结论列表）
    - NextFocus（基于任务目标推断的下一步）
    - Token 预算：≤ 500 tokens

  Step 2. 将 message_history 替换为：
    [SystemMessage: CycleBriefing 内容]
    （清除全部历史消息，仅保留检查点摘要）

  Step 3. 重置 cycle_count = 0，cycle_briefings.append(briefing)

  Step 4. 返回压缩后的 message_history 和 CheckpointResult
```

### 执行决策树

```
Compress(history) 调用时：

  始终执行 Layer 1 → history'
  if tokens(history') < threshold * 0.8:
    return history'   ← Layer 1 已足够

  match strategy:
    TrimAndProtect:
      执行 Layer 2 → history''
      return history''

    AuxiliarySummary:
      执行 Layer 2 → history''
      if still_over_threshold:
        执行 Layer 3 → history'''
      return history'''

    CycleCheckpoint:
      执行 Layer 4 → history''''（cycle_count 重置）
      return history''''
```

<!-- @end-section -->

<!-- @section: compression-trigger -->
---

## 5. 压缩触发阈值

```go
const (
    // CompressThreshold 默认压缩触发阈值：消息历史 token 数达到模型上下文窗口的 80%。
    // 不同场景可通过 AgentDef.compression_threshold 覆盖。
    CompressThreshold = 0.80

    // CycleCheckpointLength 默认循环检查点轮次。
    // 每达到此迭代轮数，ForceCheckpoint 被自动触发（不依赖 80% 阈值）。
    // 实际值来自 AgentDef.MaxCycleLength（由 CognitiveCore 在构造时传入）。
    DefaultCycleLength = 25
)
```

**SoftLoop 触发的强制检查点**：
- 当 EvalEngine 检测到 SoftLoop（ratio 5~10），AgentRuntime 立即调用 `CognitiveCore.ForceCheckpoint()`
- 此时不检查 80% 阈值，直接执行第 4 层 CycleCheckpoint
- 同时写入 `tasks.soft_loop_reset_at = now()`（时间戳用于 EvalEngine 过滤历史调用记录）

<!-- @end-section -->

<!-- @section: observability -->
---

## 6. 可观测性

每次压缩记录 `CompressionReport`，写入 `EpisodicMemoryStore`（与任务记忆一同持久化）：

| 字段 | 说明 |
|------|------|
| strategy | 使用的压缩策略 |
| layers_applied | 实际执行的层次列表 |
| tokens_before | 压缩前 token 数 |
| tokens_after | 压缩后 token 数 |
| compression_ratio | tokens_after / tokens_before |
| summary | 摘要内容（第 3/4 层时有值）|
| cycle_briefing | 检查点摘要（第 4 层时有值）|

<!-- @end-section -->
