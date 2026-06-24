---
id: "plans-agent-component-implementation-roadmap-001"
title: "Agent 组件实施路线图"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "component-roadmap", "a-components"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
related_docs:
  - path: "../../design/architecture/agent_components/index.md"
    relation: "implements"
  - path: "../../design/architecture/agent_components/agent-component-registry.md"
    relation: "implements"
---

# Agent 组件实施路线图

## 目标

依据 `agent_components/index.md` 中的 A00-A70 组件，把 Agent 独立 Go 仓库的开发计划拆成可验收的 profile、阶段和组件批次。

本路线图只规划 Agent 仓库内的实现。MaaS、Know、Common 的生产实现不在 Agent 仓库内展开；Agent 仓库只定义端口、fake adapter、recording adapter 和必要的 HTTP/gRPC client adapter。

## 装配 Profile 到开发阶段

| Profile | 阶段 | 目标 | 组件范围 |
|---------|------|------|----------|
| `minimal` | P2 | 单 Agent 可运行 | A00/A01/A02/A03/A10/A11/A20/C70/X00，A21/A22/A23/A40/A41/A63/X02 用 Noop 或 fake |
| `standard` | P3-P4 | 可观测、可治理、可用于真实任务 | minimal + A12/A21/A22/A23/A40/A41/A42/A60/A61/A63/X01/X02/X04/X05 |
| `enterprise` | P5-P6 | 技能、学习进化、多 Agent 工作流 | standard + A30/A31/A32/A43/A50/A51/A52/A53/A54/A62/A64/A70 |

## 组件实施矩阵

| ID | 组件 | 首次阶段 | Go 包 | 实施形态 | 验收要点 |
|----|------|----------|-------|----------|----------|
| A00 | CognitiveCore | P2 | `internal/cognitive` | minimal 实现 | 能组装 system/developer/task/tool/memory 上下文 |
| A01 | AgentRuntime | P2 | `internal/runtime` | minimal 实现 | 只通过 C70 推理，发布 X00 事件 |
| A02 | AgentCoordinator | P2 | `internal/runtime` | minimal 实现 | 串联调度、锁、运行、状态流转和 X02 审计 |
| A03 | ContextCompressor | P2 | `internal/cognitive` | Noop + 阈值检测 | 超过阈值时能触发摘要接口或降级 Noop |
| A10 | TaskScheduler | P2 | `internal/task` | minimal 实现 | pending/assigned/running/done/failed/suspended 流转 |
| A11 | TaskLock | P2 | `internal/task` | 完整核心 | 并发 CAS 锁定只有一个成功，过期可回收 |
| A12 | BackgroundScheduler | P3 | `internal/task` | standard 实现 | 防重入、周期任务、stale lock 回收 |
| A20 | ToolRegistry | P2 | `internal/tool` | minimal 实现 | 注册 fake tool，记录 ToolCall/ToolResult |
| A21 | ExecutionPolicy | P2 | `internal/tool` | minimal 策略 | ApprovalPolicy 与 SandboxMode 正交组合 |
| A22 | PermissionEnforcer | P2 | `internal/tool` | minimal 拦截 | company/agent/role 权限越界被拒绝 |
| A23 | ToolGuardrails | P2 | `internal/tool` | minimal 钩子 | before/after hook、重复失败计数 |
| A30 | SkillSystem | P5 | `internal/skill` | enterprise 实现 | 每任务最多注入 3 个安全技能 |
| A31 | SkillSecurityScanner | P5 | `internal/skill` | enterprise 实现 | 注入、SSRF、路径穿越规则可测 |
| A32 | SkillInstaller | P5 | `internal/skill` | enterprise 实现 | SHA-256 校验和磁盘/DB 同步 |
| A40 | MemoryProvider | P3 | `internal/memory` | standard 实现 | prefetch/system_prompt_block/sync_after_turn |
| A41 | WorkingMemory | P3 | `internal/memory` | standard 实现 | 单任务 64KB 上限、append 生命周期 |
| A42 | EpisodicMemoryStore | P3 | `internal/memory` | standard 实现 | 通过 X01 做 TopK 检索，X01 缺失时降级 |
| A43 | CapabilityMemoryStore | P5 | `internal/memory` | enterprise 实现 | Gene/Capsule 排名、注入上限、冻结 |
| A50 | SignalExtractor | P5 | `internal/evolution` | enterprise 实现 | failure/success/hardloop/feedback 信号提取 |
| A51 | GepCycle | P5 | `internal/evolution` | enterprise 实现 | scan→signal→intent→mutate→validate→solidify |
| A52 | DistillationOperator | P5 | `internal/evolution` | enterprise 实现 | Gene 六元组、token 预算、字段校验 |
| A53 | SolidifyPipeline | P5 | `internal/evolution` | enterprise 实现 | 代码/Gene 固化、爆炸半径评估 |
| A54 | EvolutionEventLog | P5 | `internal/evolution` | enterprise 实现 | 追加式进化事件，写入 X02 |
| A60 | AegisReviewer | P2 | `internal/quality` | minimal 实现 | 固定通过 C70 评审，拒绝后不得 done |
| A61 | TrustScoreManager | P4 | `internal/quality` | standard 实现 | X00/X02 事件驱动重算，三档派遣建议 |
| A62 | ApprovalService | P2/P4 | `internal/approval` | lite→full | P2 只做 HardLoop/KnowledgeReview，P4 完整门控 |
| A63 | EvalEngine | P2/P4 | `internal/quality` | lite→standard | P2 做 hardloop 检测，P4 做行为健康 Eval |
| A64 | EvolutionEvaluator | P5 | `internal/quality` | enterprise 实现 | 14 天退化检测，广播 DegradationAlert |
| A70 | WorkflowEngine | P4 | `internal/workflow` | enterprise 前置 | 8 种 DSL 原语、parallel 失败策略、ErrorHandler |

## 横向依赖落地

| 外部组件 | Agent 侧处理 |
|----------|---------------|
| C70 MaasInferenceClient | `internal/port` 定义稳定接口，P2 fake/recording，P3+ 增加 HTTP/gRPC adapter |
| X00 EventBus | P2 in-memory event bus，P3 增加订阅查询和 TUI 事件流 |
| X01 EmbeddingProvider | P3 在 memory 包通过端口接入，不直接绑定 provider |
| X02 ImmutableAuditLog | P2 in-memory/file，P4 增加不可变 hash chain adapter |
| X03 SafeFetcher | Agent 不主动实现，工具如需抓取必须通过 common 端口接入 |
| X04 PathGuard | P2 workspace-only，工具执行前强制校验 |
| X05 OutputSanitizer | P4 接入 Aegis/报告/工具输出净化 |

## 分期验收

### P2 minimal

- `agent run --demo` 通过 Bubble Tea 展示 fake task 从 pending 到 done。
- Runtime 推理只通过 C70 fake。
- 工具执行经过 A21/A22/A23。
- A60/A63-lite 能触发质量拒绝或 HardLoop。
- A62-lite 能创建工单并恢复 suspended task。

### P3 standard-memory

- A40/A41/A42 接入上下文组装。
- X01 缺失时可降级，存在时可检索情景记忆。
- A12 能回收 stale task lock。
- TUI 能展示 memory prefetch、推理事件、工具事件和审计事件。

### P4 standard-governance-workflow

- A61 信任分可影响 TaskScheduler 派遣建议。
- A62-full 覆盖预算、模型升级、危险工具、知识审核、工作流人工节点。
- A70 支持最小 DSL：sequence、parallel、approval、agent_task、error_handler。
- X02 hash chain 和 X05 输出净化接入质量与审批链。

### P5 enterprise-learning-skill

- A30/A31/A32 技能系统可安装、扫描、按任务注入。
- A43/A50-A54 跑通学习进化闭环。
- A64 能识别退化并广播告警。
- 学习资产进入 CapabilityMemoryStore 前必须经过质量、安全和审批门控。
