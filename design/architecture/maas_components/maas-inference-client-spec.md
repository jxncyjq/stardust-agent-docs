---
id: "spec-component-maas-inference-client-070"
title: "MaasInferenceClient 组件规范"
aliases: ["MaasInferenceClient规范", "MaaS推理端口", "ModelInvoker", "稳定推理端口"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "maas", "inference", "port", "facade"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C70"
layer: "L8"
depends_on:
  - "C01"   # ModelProvider — 实际同步/流式推理执行
  - "C14"   # ModelRouter — 统一模型路由决策
optional_deps:
  - "C16"   # SemanticCache — temperature=0 时可命中缓存
  - "C17"   # ContentFilter — 输入/输出内容安全过滤
  - "C19"   # TelemetryEmitter — 请求指标与链路事件
  - "C40"   # FailoverManager — 失败提供商排除与故障转移
  - "C50"   # StreamProxy — 流式响应代理
  - "C62"   # AuditLogger — MaaS 请求审计适配器
conflicts_with: []
required_by:
  - "A01"
  - "A03"
  - "A60"
  - "K13"
  - "K21"
  - "K51"
assembly_profiles:
  - minimal
  - standard
  - enterprise
  - embedded
---

# MaasInferenceClient 组件规范

## 1. 组件定位

`MaasInferenceClient` 是 MaaS 对上层模块暴露的**稳定推理端口**。Agent、Know、Aegis、SemanticExtractor、ConflictDetector 等上层组件不得直接调用 `ModelRouter`、`ModelProvider` 或具体 provider SDK，而是通过本端口提交推理请求。

它的边界是“完成一次推理调用”，内部可以组合认证上下文、路由、缓存、内容过滤、故障转移、流式代理、遥测与审计。

## 2. 职责边界

| 职责 | 归属 |
|------|------|
| 选择模型和 provider | C14 ModelRouter |
| 执行 provider 请求 | C01 ModelProvider |
| 故障转移和重试 | C40 FailoverManager / C41 RetryScheduler |
| 流式 SSE 代理 | C50 StreamProxy |
| 输入/输出安全过滤 | C17 ContentFilter |
| 语义缓存命中与写入 | C16 SemanticCache |
| 请求遥测与成本审计 | C19 TelemetryEmitter / C62 AuditLogger |
| 上层推理调用 API | C70 MaasInferenceClient |

## 3. 接口定义

```go
type MaasInferenceClient interface {
    Complete(ctx context.Context, req InferenceRequest) (InferenceResponse, error)
    Stream(ctx context.Context, req InferenceRequest, sink StreamSink) (InferenceResponse, error)
}

type InferenceRequest struct {
    RequestID   string
    TenantID    string
    UserID      string
    AgentID     string
    Purpose     InferencePurpose
    TaskType    string
    Tier        ModelTier
    ModelHint   string
    Messages    []Message
    Tools       []ToolSpec
    Temperature float64
    MaxTokens   int
    Metadata    map[string]string
}

type InferencePurpose string

const (
    PurposeAgentLoop        InferencePurpose = "agent_loop"
    PurposeContextSummary   InferencePurpose = "context_summary"
    PurposeAegisReview      InferencePurpose = "aegis_review"
    PurposeSemanticExtract  InferencePurpose = "semantic_extract"
    PurposeEntityDedup      InferencePurpose = "entity_dedup"
    PurposeConflictJudgment InferencePurpose = "conflict_judgment"
)
```

## 4. 行为契约

- 上层只传 `Purpose`、`Tier`、`TaskType`、`ModelHint` 等意图字段，不指定 provider 私有参数。
- `ModelRouter` 是端口内部依赖；上层不得把 `C14` 当作可调用 API。
- 流式调用必须通过 `Stream()`，由端口负责把 token delta 写入 `StreamSink`，并在结束时返回聚合 usage。
- 内容过滤失败时返回结构化安全错误，不把 provider 原始错误直接暴露给上层。
- `AegisReview` 默认使用 Advanced Tier，并允许被预算策略单独豁免，但仍必须记录成本和审计。
- `ContextSummary` 默认使用 Light Tier，除非上层显式传入更高 tier。
- 请求完成后必须发射遥测；审计由 `C62 AuditLogger` 作为 MaaS 请求审计适配器写入，平台级证据链仍使用 `X02 ImmutableAuditLog`。

## 5. Noop 与测试实现

| 实现 | 行为 |
|------|------|
| NoopMaasInferenceClient | 返回空文本和零 usage，仅用于单元测试 |
| FakeMaasInferenceClient | 按 `Purpose` 返回预设响应，支持 deterministic 测试 |
| RecordingMaasInferenceClient | 包装真实端口并记录请求，用于契约测试 |

生产环境不得使用 Noop 实现启动 Agent 或 Know 语义提取链路。

## 6. 使用方

| 使用方 | 调用方式 |
|--------|----------|
| A01 AgentRuntime | `Stream(PurposeAgentLoop)` 执行主循环 |
| A03 ContextCompressor | `Complete(PurposeContextSummary)` 生成摘要/检查点 |
| A60 AegisReviewer | `Complete(PurposeAegisReview)` 固定评审 |
| K13 SemanticExtractor | `Complete(PurposeSemanticExtract)` 抽取知识节点和关系 |
| K21 EntityDeduplicator | `Complete(PurposeEntityDedup)` 执行可选 LLM tiebreak |
| K51 ConflictDetector | `Complete(PurposeConflictJudgment)` 判断语义冲突 |
