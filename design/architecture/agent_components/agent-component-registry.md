---
id: "spec-agent-component-registry-000"
title: "Agent 组件注册表与依赖关系图"
aliases: ["Agent组件注册表", "agent-component-registry", "Agent依赖图"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "registry", "dependency", "assembly", "agent-engine"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
related_docs:
  - id: "architecture-agent-engine-design-nocode-001"
    relation: "derived_from"
    path: "../agent-engine-design-nocode.md"
  - id: "spec-component-registry-000"
    relation: "sibling"
    path: "../maas_components/component-registry.md"
---

<!-- @section: overview -->
# Agent 组件注册表与依赖关系图

## 文档目的

本文档是所有 Agent 引擎组件规范的**目录和依赖主表**，定义：

1. 每个组件的唯一 ID、层次归属
2. 组件间的依赖关系（必须依赖 / 可选依赖 / 互斥关系）
3. 装配方案（最小集 / 标准集 / 完整集）
4. 各组件规范文档的索引

**读者**：架构师、平台工程师、需要理解 Agent 运行时组装逻辑的实现者。

<!-- @end-section -->

<!-- @section: layer-definition -->
---

## 一、层次定义

> Agent 的 `L0~L7` 表示**能力域分组**，不是“只能向下依赖”的技术分层。核心运行时负责把任务、工具、记忆、质量、安全等能力编排到一次任务执行中，因此会通过稳定端口调用其他能力域。跨域依赖是否允许，以本注册表的“必须依赖 / 可选依赖”与 [../platform-component-registry.md](../platform-component-registry.md) 为准。

```
L0  核心编排层    — Agent 执行主链路与上下文组装
                    CognitiveCore / AgentRuntime / AgentCoordinator / ContextCompressor

L1  任务管理层    — 任务生命周期编排
                    TaskScheduler / TaskLock / BackgroundScheduler

L2  工具执行层    — 工具注册、权限、批准、沙盒
                    ToolRegistry / ExecutionPolicy / PermissionEnforcer / ToolGuardrails

L3  技能层        — 可插拔技能加载与安全扫描
                    SkillSystem / SkillSecurityScanner / SkillInstaller

L4  记忆层        — 三层记忆系统（工作/情景/能力）
                    MemoryProvider / WorkingMemory / EpisodicMemoryStore / CapabilityMemoryStore

L5  学习进化层    — GEP 进化周期与资产管理
                    SignalExtractor / GepCycle / DistillationOperator / SolidifyPipeline / EvolutionEventLog

L6  质量安全层    — 质量门控、信任评分、行为评估
                    AegisReviewer / TrustScoreManager / ApprovalService / EvalEngine / EvolutionEvaluator

L7  工作流层      — 多 Agent 协作 DSL 执行引擎
                    WorkflowEngine

公共层（X）       — 与 MaaS / Know 共用的基础设施
                    EventBus / EmbeddingProvider / ImmutableAuditLog / SafeFetcher / PathGuard / OutputSanitizer

MaaS端口（C70）   — 上层调用 MaaS 的稳定推理门面
                    MaasInferenceClient
```

**依赖约束**：

1. 禁止循环依赖。
2. Agent 组件不得直接依赖 `C01 ModelProvider`、具体 provider SDK 或 `C14 ModelRouter`。
3. 需要 LLM 推理时统一通过 `C70 MaasInferenceClient`。
4. `A02 AgentCoordinator` 可以编排 `A10 TaskScheduler`、`A61 TrustScoreManager`、`A62 ApprovalService` 等能力，但这些是编排依赖，不代表“低层依赖高层”。

<!-- @end-section -->

<!-- @section: component-table -->
---

## 二、组件主表

### L0 核心运行时

| ID  | 组件 | 层 | 必须依赖 | 可选依赖 | 规范文档 |
|-----|------|----|---------|---------|---------|
| A00 | CognitiveCore | L0 | — | A40(MemoryProvider), A30(SkillSystem) | [cognitive-core-spec.md](./cognitive-core-spec.md) |
| A01 | AgentRuntime | L0 | A00, A20, C70, X00 | A63(EvalEngine), A22(PermissionEnforcer), A23(ToolGuardrails) | [agent-runtime-spec.md](./agent-runtime-spec.md) |
| A02 | AgentCoordinator | L0 | A01, A10, A11, X02 | A61(TrustScoreManager), A62(ApprovalService) | [agent-coordinator-spec.md](./agent-coordinator-spec.md) |
| A03 | ContextCompressor | L0 | — | C70(MaasInferenceClient) | [context-compressor-spec.md](./context-compressor-spec.md) |

> A03 由 A00 持有所有权，不独立注册，但规范独立以便实现者参考。

### L1 任务管理

| ID  | 组件 | 层 | 必须依赖 | 可选依赖 | 规范文档 |
|-----|------|----|---------|---------|---------|
| A10 | TaskScheduler | L1 | A11 | — | [task-scheduler-spec.md](./task-scheduler-spec.md) |
| A11 | TaskLock | L1 | — | — | [task-lock-spec.md](./task-lock-spec.md) |
| A12 | BackgroundScheduler | L1 | — | — | [background-scheduler-spec.md](./background-scheduler-spec.md) |

### L2 工具执行

| ID  | 组件 | 层 | 必须依赖 | 可选依赖 | 规范文档 |
|-----|------|----|---------|---------|---------|
| A20 | ToolRegistry | L2 | — | — | [tool-registry-spec.md](./tool-registry-spec.md) |
| A21 | ExecutionPolicy | L2 | — | — | [execution-policy-spec.md](./execution-policy-spec.md) |
| A22 | PermissionEnforcer | L2 | — | — | [permission-enforcer-spec.md](./permission-enforcer-spec.md) |
| A23 | ToolGuardrails | L2 | — | — | [tool-guardrails-spec.md](./tool-guardrails-spec.md) |

### L3 技能

| ID  | 组件 | 层 | 必须依赖 | 可选依赖 | 规范文档 |
|-----|------|----|---------|---------|---------|
| A30 | SkillSystem | L3 | A31 | A32 | [skill-system-spec.md](./skill-system-spec.md) |
| A31 | SkillSecurityScanner | L3 | — | — | [skill-security-scanner-spec.md](./skill-security-scanner-spec.md) |
| A32 | SkillInstaller | L3 | A31 | — | [skill-installer-spec.md](./skill-installer-spec.md) |

### L4 记忆

| ID  | 组件 | 层 | 必须依赖 | 可选依赖 | 规范文档 |
|-----|------|----|---------|---------|---------|
| A40 | MemoryProvider | L4 | A42 | A43, X01 | [memory-provider-spec.md](./memory-provider-spec.md) |
| A41 | WorkingMemory | L4 | — | — | [working-memory-spec.md](./working-memory-spec.md) |
| A42 | EpisodicMemoryStore | L4 | X01 | — | [episodic-memory-store-spec.md](./episodic-memory-store-spec.md) |
| A43 | CapabilityMemoryStore | L4 | — | X01 | [capability-memory-store-spec.md](./capability-memory-store-spec.md) |

### L5 学习进化

| ID  | 组件 | 层 | 必须依赖 | 可选依赖 | 规范文档 |
|-----|------|----|---------|---------|---------|
| A50 | SignalExtractor | L5 | — | — | [signal-extractor-spec.md](./signal-extractor-spec.md) |
| A51 | GepCycle | L5 | A50, A52, A53, A54, A43 | — | [gep-cycle-spec.md](./gep-cycle-spec.md) |
| A52 | DistillationOperator | L5 | — | — | [distillation-operator-spec.md](./distillation-operator-spec.md) |
| A53 | SolidifyPipeline | L5 | A43, A54 | — | [solidify-pipeline-spec.md](./solidify-pipeline-spec.md) |
| A54 | EvolutionEventLog | L5 | X02 | — | [evolution-event-log-spec.md](./evolution-event-log-spec.md) |

### L6 质量安全

| ID  | 组件 | 层 | 必须依赖 | 可选依赖 | 规范文档 |
|-----|------|----|---------|---------|---------|
| A60 | AegisReviewer | L6 | C70, X00 | — | [aegis-reviewer-spec.md](./aegis-reviewer-spec.md) |
| A61 | TrustScoreManager | L6 | X00, X02 | — | [trust-score-manager-spec.md](./trust-score-manager-spec.md) |
| A62 | ApprovalService | L6 | A10, X00 | — | [approval-service-spec.md](./approval-service-spec.md) |
| A63 | EvalEngine | L6 | — | — | [eval-engine-spec.md](./eval-engine-spec.md) |
| A64 | EvolutionEvaluator | L6 | X00 | A63 | [evolution-evaluator-spec.md](./evolution-evaluator-spec.md) |

### L7 工作流

| ID  | 组件 | 层 | 必须依赖 | 可选依赖 | 规范文档 |
|-----|------|----|---------|---------|---------|
| A70 | WorkflowEngine | L7 | A10, A62, X00 | — | [workflow-engine-spec.md](./workflow-engine-spec.md) |

### 公共组件（来自 common_components）

| ID  | 组件 | 规范文档 |
|-----|------|---------|
| X00 | EventBus | [../common_components/event-bus-spec.md](../common_components/event-bus-spec.md) |
| X01 | EmbeddingProvider | [../common_components/embedding-provider-spec.md](../common_components/embedding-provider-spec.md) |
| X02 | ImmutableAuditLog | [../common_components/immutable-audit-log-spec.md](../common_components/immutable-audit-log-spec.md) |
| X03 | SafeFetcher | [../common_components/safe-fetcher-spec.md](../common_components/safe-fetcher-spec.md) |
| X04 | PathGuard | [../common_components/path-guard-spec.md](../common_components/path-guard-spec.md) |
| X05 | OutputSanitizer | [../common_components/output-sanitizer-spec.md](../common_components/output-sanitizer-spec.md) |

### MaaS 稳定端口

| ID  | 组件 | 规范文档 |
|-----|------|---------|
| C70 | MaasInferenceClient | [../maas_components/maas-inference-client-spec.md](../maas_components/maas-inference-client-spec.md) |

<!-- @end-section -->

<!-- @section: dependency-graph -->
---

## 三、关键依赖关系图

```
                    ┌─────────────────────────────────────────────────────┐
                    │                   L7 WorkflowEngine (A70)            │
                    └────────────────────┬────────────────────────────────┘
                                         │ depends on
                    ┌────────────────────▼────────────────────────────────┐
                    │              L6 质量安全层                           │
                    │  AegisReviewer(A60)  TrustScoreManager(A61)         │
                    │  ApprovalService(A62)  EvalEngine(A63)              │
                    │  EvolutionEvaluator(A64)                            │
                    └────────────────────┬────────────────────────────────┘
                                         │
                    ┌────────────────────▼────────────────────────────────┐
                    │              L5 学习进化层                           │
                    │  SignalExtractor(A50) → GepCycle(A51)               │
                    │  DistillationOperator(A52) SolidifyPipeline(A53)    │
                    │  EvolutionEventLog(A54)                             │
                    └────────────────────┬────────────────────────────────┘
                                         │
         ┌───────────────────────────────▼───────────────────────────────┐
         │                         L4 记忆层                              │
         │   MemoryProvider(A40) → EpisodicMemoryStore(A42)              │
         │   WorkingMemory(A41)    CapabilityMemoryStore(A43)             │
         └─────────────────────────┬─────────────────────────────────────┘
                                   │
    ┌──────────────────────────────▼──────────────────────────────────┐
    │                     L0~L3 核心执行层                              │
    │                                                                   │
    │  AgentCoordinator(A02)                                            │
    │    ├── AgentRuntime(A01)                                          │
    │    │     ├── CognitiveCore(A00)                                   │
    │    │     │     └── ContextCompressor(A03)                         │
    │    │     ├── ToolRegistry(A20)                                    │
    │    │     │     ├── ExecutionPolicy(A21)                           │
    │    │     │     ├── PermissionEnforcer(A22)                        │
    │    │     │     └── ToolGuardrails(A23)                            │
    │    │     └── EvalEngine(A63) ← 实时收敛比检测                     │
    │    ├── TaskScheduler(A10) ── TaskLock(A11)                        │
    │    └── BackgroundScheduler(A12)                                   │
    │                                                                   │
    │  SkillSystem(A30) ── SkillSecurityScanner(A31)                    │
    │                   └── SkillInstaller(A32)                         │
    └──────────────────────────────┬──────────────────────────────────┘
                                   │
    ┌──────────────────────────────▼──────────────────────────────────┐
    │              公共基础设施层（common_components）                   │
    │  EventBus(X00)   EmbeddingProvider(X01)   ImmutableAuditLog(X02) │
    │  SafeFetcher(X03) PathGuard(X04) OutputSanitizer(X05)            │
    └────────────────────────────────────────────────────────────────┘

    ┌────────────────────────────────────────────────────────────────┐
    │              MaaS 推理端口                                      │
    │  MaasInferenceClient(C70)                                      │
    └────────────────────────────────────────────────────────────────┘
```

<!-- @end-section -->

<!-- @section: assembly-profiles -->
---

## 四、装配方案（Assembly Profiles）

### minimal（最小可运行集）

Agent 可接收任务并执行 LLM 调用，无记忆、无学习、无质量门控：

```
必须：A00, A01, A02, A03, A10, A11, A20, C70, X00
可选（降级 Noop）：A21, A22, A23, A40, A41, A63, X02
```

### standard（标准生产集）

增加工具安全、记忆检索、基础评估：

```
= minimal
+ A21(ExecutionPolicy), A22(PermissionEnforcer), A23(ToolGuardrails)
+ A40(MemoryProvider), A41(WorkingMemory), A42(EpisodicMemoryStore)
+ A12(BackgroundScheduler)
+ A60(AegisReviewer), A61(TrustScoreManager), A63(EvalEngine)
+ X01(EmbeddingProvider), X02(ImmutableAuditLog)
```

### enterprise（完整功能集）

增加技能系统、GEP 进化、工作流 DSL：

```
= standard
+ A30, A31, A32（技能系统）
+ A43(CapabilityMemoryStore)
+ A50~A54（学习进化全链路）
+ A62(ApprovalService), A64(EvolutionEvaluator)
+ A70(WorkflowEngine)
```

<!-- @end-section -->

<!-- @section: noop-behaviors -->
---

## 五、Noop 降级行为

可选依赖缺失时，框架注入 Noop 实现，保证 Agent 正常启动：

| 缺失组件 | Noop 行为 |
|---------|----------|
| MemoryProvider（A40） | `prefetch` 返回空、`system_prompt_block` 返回空字符串 |
| EvalEngine（A63） | `eval_trace()` 始终返回 `Normal`，永不触发循环检测 |
| TrustScoreManager（A61） | `effective_score()` 始终返回 1.0，跳过信任约束 |
| ApprovalService（A62） | 所有工具调用视为已批准，不创建审批工单 |
| SkillSystem（A30） | `load_for_task()` 返回空技能列表 |
| ImmutableAuditLog（X02） | 审计条目写入 stdout，不持久化（仅开发环境可用） |
| EmbeddingProvider（X01） | 向量检索返回空结果，仅使用 FTS5 全文检索 |

<!-- @end-section -->
