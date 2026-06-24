---
id: "index-agent-components"
title: "Agent 引擎组件规范索引"
aliases: ["Agent组件索引", "agent-components-index"]
type: "index"
category: "design/architecture/agent_components"
tags: ["index", "component-spec", "agent-engine", "legion"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "active"
related_docs:
  - id: "architecture-agent-engine-design-nocode-001"
    relation: "derived_from"
    path: "../agent-engine-design-nocode.md"
  - id: "index-components"
    relation: "sibling"
    path: "../maas_components/index.md"
---

# Legion Agent 引擎组件规范索引

本目录收录 Legion **Agent 引擎**的全部组件规范文档，对应 `legion-core`（Layer 3）及其所依赖的 Layer 0~2 专属子系统。

依赖关系总览、装配方案（Profile）请查阅主表文档：
→ **[[agent-component-registry|Agent 组件注册表与依赖关系图]]**

全平台 A/C/K/X 跨模块依赖请查阅：
→ **[[../platform-component-registry|平台组件注册表与跨模块依赖]]**

MaaS 层组件索引（`legion-maas`）请查阅：
→ **[[../maas_components/index|MaaS 组件规范索引]]**

MaaS 稳定推理端口（MaasInferenceClient / C70）请查阅：
→ **[[../maas_components/maas-inference-client-spec|MaasInferenceClient 组件规范]]**

三层共用的基础组件（EventBus / EmbeddingProvider / ImmutableAuditLog / SafeFetcher / PathGuard / OutputSanitizer）请查阅：
→ **[[../common_components/index|通用组件规范索引]]**

---

## 层次概览

```
L0  核心编排层    — CognitiveCore / AgentRuntime / AgentCoordinator / ContextCompressor
L1  任务管理层    — TaskScheduler / TaskLock / BackgroundScheduler
L2  工具执行层    — ToolRegistry / ExecutionPolicy / PermissionEnforcer / ToolGuardrails
L3  技能层        — SkillSystem / SkillSecurityScanner / SkillInstaller
L4  记忆层        — MemoryProvider / WorkingMemory / EpisodicMemoryStore / CapabilityMemoryStore
L5  学习进化层    — SignalExtractor / GepCycle / DistillationOperator / SolidifyPipeline / EvolutionEventLog
L6  质量安全层    — AegisReviewer / TrustScoreManager / ApprovalService / EvalEngine / EvolutionEvaluator
L7  工作流层      — WorkflowEngine
```

---

> `L0~L7` 是能力域分组，不是强制依赖方向。核心编排层可以通过稳定端口编排任务、工具、质量、安全等能力；禁止的是循环依赖和直接绑定 provider 细节。

---

## 核心编排层（L0）

| ID  | 组件 | 规范文档 | 说明 |
|-----|------|----------|------|
| A00 | CognitiveCore | [cognitive-core-spec.md](./cognitive-core-spec.md) | 7 要素上下文组装、优先级编排（P0~P6）、系统提示缓存锁定 |
| A01 | AgentRuntime | [agent-runtime-spec.md](./agent-runtime-spec.md) | LLM 调用主循环、工具执行链、收敛比实时检测、学习事件发布 |
| A02 | AgentCoordinator | [agent-coordinator-spec.md](./agent-coordinator-spec.md) | 心跳协议九步流程、原子锁定、路由决策、任务状态流转、审计链写入 |
| A03 | ContextCompressor | [context-compressor-spec.md](./context-compressor-spec.md) | 四层压缩策略（裁剪/保护/摘要/检查点）、80% 触发阈值、循环检查点 |

---

## 任务管理层（L1）

| ID  | 组件 | 规范文档 | 说明 |
|-----|------|----------|------|
| A10 | TaskScheduler | [task-scheduler-spec.md](./task-scheduler-spec.md) | 七状态任务机、inbox→assigned 派遣、stale 回收、硬约束保护 |
| A11 | TaskLock | [task-lock-spec.md](./task-lock-spec.md) | 原子 CAS 锁定、expires_at 过期、并发安全保证 |
| A12 | BackgroundScheduler | [background-scheduler-spec.md](./background-scheduler-spec.md) | 防重入 AtomicBool、分散初始延迟、动态启用开关 |

---

## 工具执行层（L2）

| ID  | 组件 | 规范文档 | 说明 |
|-----|------|----------|------|
| A20 | ToolRegistry | [tool-registry-spec.md](./tool-registry-spec.md) | 工具注册、路由、并发执行、MCP 代理工具 |
| A21 | ExecutionPolicy | [execution-policy-spec.md](./execution-policy-spec.md) | ApprovalPolicy × SandboxMode 正交组合、RBAC 角色覆盖、auto_allow 前缀 |
| A22 | PermissionEnforcer | [permission-enforcer-spec.md](./permission-enforcer-spec.md) | 批量权限检查、越权拦截、来自 Claw Code |
| A23 | ToolGuardrails | [tool-guardrails-spec.md](./tool-guardrails-spec.md) | before/after 调用钩子、重复失败检测、工具执行看门狗 |

---

## 技能层（L3）

| ID  | 组件 | 规范文档 | 说明 |
|-----|------|----------|------|
| A30 | SkillSystem | [skill-system-spec.md](./skill-system-spec.md) | 7 类来源扫描、去重策略、按任务运行时注入（上限 3 个/任务） |
| A31 | SkillSecurityScanner | [skill-security-scanner-spec.md](./skill-security-scanner-spec.md) | 13 条规则、Critical/Warning/Info 三级、注入/SSRF/路径穿越检测 |
| A32 | SkillInstaller | [skill-installer-spec.md](./skill-installer-spec.md) | 注册表拉取（ClawdHub/skills.sh）、SHA-256 完整性、磁盘↔DB 双向同步 |

---

## 记忆层（L4）

| ID  | 组件 | 规范文档 | 说明 |
|-----|------|----------|------|
| A40 | MemoryProvider | [memory-provider-spec.md](./memory-provider-spec.md) | 三层记忆统一接口（system_prompt_block / prefetch / sync_after_turn） |
| A41 | WorkingMemory | [working-memory-spec.md](./working-memory-spec.md) | 运行时草稿本、64KB 上限、append 模式、单次任务生命周期 |
| A42 | EpisodicMemoryStore | [episodic-memory-store-spec.md](./episodic-memory-store-spec.md) | SQLite 持久化情景记忆、余弦相似度检索（TopK=5）、跨任务 |
| A43 | CapabilityMemoryStore | [capability-memory-store-spec.md](./capability-memory-store-spec.md) | Gene/Capsule 能力资产存储、成功率排名、注入上限管控 |

> 向量嵌入能力由共用组件 EmbeddingProvider（X01）提供，见 [[../common_components/index]]。

---

## 学习进化层（L5）

| ID  | 组件 | 规范文档 | 说明 |
|-----|------|----------|------|
| A50 | SignalExtractor | [signal-extractor-spec.md](./signal-extractor-spec.md) | 三层信号提取（正则/关键词/LLM）、信号抑制（8周期≥3次）、ForceInnovate |
| A51 | GepCycle | [gep-cycle-spec.md](./gep-cycle-spec.md) | GEP 六阶段形式化周期（scan→signal→intent→mutate→validate→solidify） |
| A52 | DistillationOperator | [distillation-operator-spec.md](./distillation-operator-spec.md) | 蒸馏算子 ψ(s)/ψ(ℋ)、Gene 六元组、230 token 预算、α 字段强制非空 |
| A53 | SolidifyPipeline | [solidify-pipeline-spec.md](./solidify-pipeline-spec.md) | 代码变更固化（6步）/ Gene 资产固化（5步）、爆炸半径评估、Ed25519 封印 |
| A54 | EvolutionEventLog | [evolution-event-log-spec.md](./evolution-event-log-spec.md) | 不可变进化审计链、九元组 EvolutionEvent、Ed25519 immutable_seal |

> CapabilityMemoryStore（A43）同时承担 Gene/Capsule 持久化，学习层通过 Arc 共享同一实例。

---

## 质量安全层（L6）

| ID  | 组件 | 规范文档 | 说明 |
|-----|------|----------|------|
| A60 | AegisReviewer | [aegis-reviewer-spec.md](./aegis-reviewer-spec.md) | 固定 LLM 质量评审、max_rejection_cycles=3、Advanced Tier 豁免预算降级 |
| A61 | TrustScoreManager | [trust-score-manager-spec.md](./trust-score-manager-spec.md) | 事件驱动实时重算、初始 0.7、冷却期懒惰求值、派遣决策三档 |
| A62 | ApprovalService | [approval-service-spec.md](./approval-service-spec.md) | 七大审批门控、工单创建、resume_task 恢复链、Allowlist CAS 并发 |
| A63 | EvalEngine | [eval-engine-spec.md](./eval-engine-spec.md) | 四层行为健康 Eval（输出/追踪/组件/漂移）、收敛比实时检测、8周趋势 |
| A64 | EvolutionEvaluator | [evolution-evaluator-spec.md](./evolution-evaluator-spec.md) | 5 维度能力评估、14 天退化检测（双条件）、DegradationAlert 广播 |

---

## 工作流层（L7）

| ID  | 组件 | 规范文档 | 说明 |
|-----|------|----------|------|
| A70 | WorkflowEngine | [workflow-engine-spec.md](./workflow-engine-spec.md) | 8 种 DSL 原语执行引擎、Parallel 失败策略、ErrorHandler、工作流状态机 |

---

## 通用组件（在 common_components 目录）

以下组件被 Agent 引擎、MaaS 层与 Know 层共同依赖，规范在独立目录维护：

| ID  | 组件 | 规范文档 | 使用方 |
|-----|------|----------|--------|
| X00 | EventBus | [../common_components/event-bus-spec.md](../common_components/event-bus-spec.md) | AgentRuntime（SSE token 流）、TrustScoreManager（安全事件广播）、AegisReviewer（审核结果推送） |
| X01 | EmbeddingProvider | [../common_components/embedding-provider-spec.md](../common_components/embedding-provider-spec.md) | EpisodicMemoryStore（记忆检索）、MaaS SemanticCache、Know VectorStore |
| X02 | ImmutableAuditLog | [../common_components/immutable-audit-log-spec.md](../common_components/immutable-audit-log-spec.md) | AgentCoordinator（任务审计链）、EvolutionEventLog（进化审计）、KnowledgeGovernance |
| X03 | SafeFetcher | [../common_components/safe-fetcher-spec.md](../common_components/safe-fetcher-spec.md) | Know ContentIngestor 外部 URL 接入，Agent 工具可复用安全抓取策略 |
| X04 | PathGuard | [../common_components/path-guard-spec.md](../common_components/path-guard-spec.md) | Agent 工具沙盒、Know 语料扫描、MCP/导出路径约束 |
| X05 | OutputSanitizer | [../common_components/output-sanitizer-spec.md](../common_components/output-sanitizer-spec.md) | Aegis/SemanticExtractor/报告/MCP 输出净化 |

---

## MaaS 推理端口

| ID | 组件 | 规范文档 | 使用方 |
|----|------|----------|--------|
| C70 | MaasInferenceClient | [../maas_components/maas-inference-client-spec.md](../maas_components/maas-inference-client-spec.md) | AgentRuntime 主循环、ContextCompressor 摘要、AegisReviewer 质量评审 |

---

## 文档统计

| 层次 | 组件数 | 独立规范数 | 说明 |
|------|--------|-----------|------|
| L0 核心编排 | 4 | 4 | |
| L1 任务管理 | 3 | 3 | |
| L2 工具执行 | 4 | 4 | |
| L3 技能 | 3 | 3 | |
| L4 记忆 | 4 | 4 | |
| L5 学习进化 | 5 | 5 | |
| L6 质量安全 | 5 | 5 | |
| L7 工作流 | 1 | 1 | |
| **小计** | **29** | **29** | |
| 通用组件（common） | 6 | 6 | 另见 common_components/ |
| MaaS 推理端口 | 1 | 1 | 另见 maas_components/ |
| **总计** | **36** | **36** | |

---

## 快速入口

| 需求 | 文档 |
|------|------|
| 了解 Agent 完整依赖图 | [agent-component-registry.md](./agent-component-registry.md) |
| Agent 任务执行全链路 | [agent-coordinator-spec.md](./agent-coordinator-spec.md) · [agent-runtime-spec.md](./agent-runtime-spec.md) |
| 上下文与记忆管理 | [cognitive-core-spec.md](./cognitive-core-spec.md) · [context-compressor-spec.md](./context-compressor-spec.md) · [memory-provider-spec.md](./memory-provider-spec.md) |
| 工具执行与安全 | [tool-registry-spec.md](./tool-registry-spec.md) · [execution-policy-spec.md](./execution-policy-spec.md) · [permission-enforcer-spec.md](./permission-enforcer-spec.md) |
| 学习与进化 | [gep-cycle-spec.md](./gep-cycle-spec.md) · [distillation-operator-spec.md](./distillation-operator-spec.md) · [solidify-pipeline-spec.md](./solidify-pipeline-spec.md) |
| 质量门控与信任 | [aegis-reviewer-spec.md](./aegis-reviewer-spec.md) · [trust-score-manager-spec.md](./trust-score-manager-spec.md) · [approval-service-spec.md](./approval-service-spec.md) |
| 任务调度与后台 | [task-scheduler-spec.md](./task-scheduler-spec.md) · [background-scheduler-spec.md](./background-scheduler-spec.md) |
| 工作流 DSL | [workflow-engine-spec.md](./workflow-engine-spec.md) |
