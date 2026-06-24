---
id: "plans-agent-task-breakdown-001"
title: "Agent 开发任务详情表"
type: "task-breakdown"
category: "plans/agent"
tags: ["plan", "agent", "task-table", "execution"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-06-01"
status: "active"
related_docs:
  - path: "./index.md"
    relation: "derived_from"
  - path: "./component-implementation-roadmap.md"
    relation: "derived_from"
  - path: "./sprint-01-scaffold.md"
    relation: "derived_from"
---

# Agent 开发任务详情表

## 状态说明

| 状态 | 含义 |
|------|------|
| `done` | 已有代码并通过当前验证 |
| `partial` | 已有最小代码，但未达到计划完整验收 |
| `todo` | 尚未开始编码 |
| `blocked` | 依赖外部条件或上游组件 |

## 当前进度摘要

| 阶段 | 目标 | 当前状态 | 说明 |
|------|------|----------|------|
| P2 minimal | 单 Agent 可运行闭环 | `done` | A10/A02/A00/A03/A63-lite/X04 与 task/model/tool/review/approval 审计链已补齐 |
| P3 standard-memory | 记忆、后台调度、事件流 | `done` | BackgroundScheduler、MemoryProvider、WorkingMemory、EpisodicMemoryStore、TUI 事件流和 SQLite 本地持久化已完成 |
| P4 governance-workflow | 信任、完整审批、工作流 | `done` | TrustScoreManager、A62-full、Eval standard、WorkflowEngine MVP、X02 hash chain、X05 OutputSanitizer 已完成 |
| P5 enterprise-learning-skill | 技能、学习进化、退化检测 | `done` | SkillSystem、SkillSecurityScanner、SkillInstaller、CapabilityMemoryStore、SignalExtractor、GEP 学习闭环、EvolutionEvaluator 已完成 |
| P6 integration | 跨组件集成补强 | `done` | CognitiveCore 注入技能/能力记忆、学习事件、GEP 后台触发、P5 SQLite 持久化、退化治理、wait_event/subworkflow 已完成 |
| P7 production-readiness | 真实适配器、恢复能力和端到端操作面 | `done` | MaaS HTTP 适配器、SQLite 跨进程恢复、CLI real run mode、端到端 smoke 入口已完成 |
| P8 service-readiness | 服务化入口、配置、API 与 CI | `done` | 配置加载、服务化启动、HTTP API、持久化运行模式、CI 发布流水线已完成 |
| P9 operability-release | 运维化、观测性、安全边界与发布准备 | `done` | HTTP 管理令牌、请求追踪、结构化日志、metrics、readiness、diagnostics、SQLite 版本/备份/恢复、release build、compat 门禁、runbook 已完成 |
| P10 component-parity | 对齐 Agent 组件规范高级行为 | `done` | parity gate、A03 四层压缩、A20-A23 工具流水线、A30-A32 Skill lifecycle、A52-A54 Gene 固化、A60-A64 历史化、A70 高级工作流和总验收已完成 |
| P11 platform-integration | 平台集成就绪 | `done` | OpenAPI 契约、X00/SSE 事件流、tenant/company 边界、Prometheus 出口、数据保留与归档已完成 |
| P12 enterprise-governance-ecosystem | 企业治理与外部生态硬化 | `done` | RBAC audit/quality 查询边界、远端 Skill registry、MaaS profile、trace 出口、OpenAPI 错误契约、CI 门禁与总验收已完成 |
| P13 runtime-context-memory | 运行时上下文文件与记忆目录 | `done` | SOUL/AGENTS/TOOLS/USER/MEMORY 加载、docs/memory 工作空间约束、CLI 开关和专项测试已完成 |
| P14 runtime-cognitive-integration | Runtime 与 CognitiveCore 集成 | `done` | A01 已通过 A00 组装 prompt 后调用 C70，P13 上下文文件经 CognitiveCore 注入 |
| P15 runtime-interrupt | Runtime 中断控制 | `done` | Interrupt 控制面、interrupted 轻量学习事件和中断错误已完成 |
| P16 runtime-default-ports | Runtime 默认端口与缺失依赖保护 | `done` | EventBus/AuditLog 缺失时 Noop 降级，MaaS 缺失时返回可匹配错误 |
| P17 interactive-tui | 交互式 TUI | `done` | 新增 `agent tui`，支持输入 prompt、运行任务、结果展示和五类上下文文件加载 |
| P18 tui-visual-polish | TUI 视觉与交互增强 | `done` | `agent tui` 使用 alternate screen 清屏进入，并展示 Bubble Tea 分区面板、状态栏和帮助栏 |
| P19 tui-workbench-layout | TUI 工作台布局 | `done` | 参考 DeepSeek-TUI 截图构建 header/main/plan/composer/footer 全屏界面 |
| P20 multi-agent-runtime-routing | 多 Agent Runtime 路由 | `done` | 已按 `task.agent_id` 路由到不同 agent 配置、模型 profile、上下文和 workspace |
| P21 agent-message-bus | Agent 通讯与消息总线 | `done` | P21A TaskLedger、SQLite inbox/outbox、send/read message 工具、TUI `/send` `/inbox`、`@agent --inbox`、workflow result handoff、message HTTP API 和协作文档/兼容性测试已完成 |
| P22 session-context-continuity | 会话上下文连续性 | `done` | 已补齐 AgentSession/ConversationTurn、最近 N 轮上下文注入、TUI session 命令和 HTTP session 查询接口 |
| P24 role-scoped-skills | 分角色 Skill 管理与注入 | `done` | 已实现：角色级 `skills.install_root` 参与 Runtime prompt 注入，CLI/TUI skill 管理支持选择 agent |

## P2 minimal 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P2-001 | P0 | Repo | 初始化 Go 1.26.0 独立仓库 | `done` | `legion/legionAgent/go.mod` | `go version` 为 1.26.0；`go test ./...` 可运行 |
| AG-P2-002 | P0 | CLI | 创建服务入口与 Cobra 命令 | `done` | `legion/legionAgent/cmd/main.go`, `internal/cli` | `go run ./cmd -- run --demo --plain` 输出 demo 结果 |
| AG-P2-003 | P0 | TUI | Bubble Tea 终端 UI 骨架 | `done` | `legion/legionAgent/internal/tui` | `agent run --demo` 可进入 TUI；plain 模式可用于 CI |
| AG-P2-004 | P0 | Domain | 定义 Agent/Task/TaskRun/Tool/Audit 领域对象 | `done` | `legion/legionAgent/internal/domain` | 编译通过，Runtime/Tool/Quality 复用同一领域模型 |
| AG-P2-005 | P0 | C70/X00/X02 | 定义外部端口和 fake adapter | `done` | `internal/port`, `internal/adapter` | Runtime 只通过 `MaasInferenceClient` 调模型 |
| AG-P2-006 | P0 | A11 | TaskLock 并发锁 | `done` | `internal/task/lock.go` | 并发 `TryLock` 只有一个成功 |
| AG-P2-007 | P0 | A20/A21/A22/A23 | ToolRegistry、策略、权限、guardrails 最小实现 | `done` | `internal/tool/registry.go` | 被拒绝权限不会调用工具 handler |
| AG-P2-008 | P0 | A62-lite | HardLoop/KnowledgeReview 最小审批工单 | `done` | `internal/approval/service.go` | 工单可创建、批准、拒绝 |
| AG-P2-009 | P0 | A60 | AegisReviewer 最小质量评审 | `done` | `internal/quality/reviewer.go` | unsafe 输出被拒绝 |
| AG-P2-010 | P0 | A01 | AgentRuntime fake task 主循环 | `done` | `internal/runtime/runtime.go` | fake task 调 C70、写事件、写审计并返回结果 |
| AG-P2-011 | P0 | Build/Test | Makefile 与基础验证 | `done` | `legion/legionAgent/Makefile` | `go test ./...`、`go vet ./...`、`go build -o NUL ./cmd` 通过 |
| AG-P2-012 | P0 | A10 | TaskScheduler 七状态任务机 | `done` | `internal/task` | pending→assigned→running→quality_review→done；suspended/failed 分支可测 |
| AG-P2-013 | P0 | A02 | AgentCoordinator 九步执行链 | `done` | `internal/runtime` | 串联 scheduler、lock、runtime、quality、approval、audit |
| AG-P2-014 | P0 | A00 | CognitiveCore 上下文组装 | `done` | `internal/cognitive` | 组装 system/developer/task/tool/memory 上下文块 |
| AG-P2-015 | P1 | A03 | ContextCompressor Noop + 阈值检测 | `done` | `internal/cognitive` | 超阈值触发压缩策略或 Noop 降级事件 |
| AG-P2-016 | P1 | A63-lite | HardLoop / convergence 检测 | `done` | `internal/quality` | 循环触发 suspended 和 A62-lite 工单 |
| AG-P2-017 | P1 | X04 | PathGuard workspace-only 端口 | `done` | `internal/port`, `internal/tool` | 工具路径越界被拒绝 |
| AG-P2-018 | P1 | Audit | 审计 request_id 串联 | `done` | `internal/runtime`, `internal/adapter`, `internal/tool` | task/model/tool/review/approval 可追踪 |

## P3 standard-memory 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P3-001 | P0 | A12 | BackgroundScheduler | `done` | `internal/task` | 防重入、周期调度、stale lock 回收 |
| AG-P3-002 | P0 | A40 | MemoryProvider 接口与 Noop/standard 实现 | `done` | `internal/memory`, `internal/cognitive` | `prefetch`、`system_prompt_block`、`sync_after_turn` 可测 |
| AG-P3-003 | P0 | A41 | WorkingMemory | `done` | `internal/memory` | 单任务 append，64KB 上限，任务结束释放 |
| AG-P3-004 | P0 | A42/X01 | EpisodicMemoryStore + EmbeddingProvider fake | `done` | `internal/memory`, `internal/port`, `internal/adapter` | TopK 检索；X01 缺失时降级 |
| AG-P3-005 | P1 | Event Stream | TUI 事件流增强 | `done` | `internal/tui`, `internal/app`, `internal/runtime`, `internal/adapter` | 展示 memory prefetch、推理、工具、审计事件 |
| AG-P3-006 | P1 | Storage | SQLite repository | `done` | `internal/storage` | task、lock、run、audit 本地演示可持久化 |

## P4 governance-workflow 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P4-001 | P0 | A61 | TrustScoreManager | `done` | `internal/quality` | 初始 0.7，事件驱动重算，allow/cautious/blocked |
| AG-P4-002 | P0 | A62-full | 完整审批门控 | `done` | `internal/approval` | 七类门控：HardLoop、KnowledgeReview、BudgetExceeded、ModelUpgrade、DangerousTool、WorkflowHumanGate、SkillInstall |
| AG-P4-003 | P0 | A63 | EvalEngine standard | `done` | `internal/quality` | 输出/追踪/组件/漂移四层行为健康 Eval |
| AG-P4-004 | P0 | A70 | WorkflowEngine MVP DSL | `done` | `internal/workflow` | sequence、parallel、agent_task、approval、condition、error_handler |
| AG-P4-005 | P1 | X02 | ImmutableAuditLog hash chain adapter | `done` | `internal/adapter` | 审计只追加，hash chain 可校验 |
| AG-P4-006 | P1 | X05 | OutputSanitizer | `done` | `internal/port`, `internal/quality`, `internal/tool`, `internal/tui` | Aegis 结论、工具输出、TUI 展示前净化 |

## P5 enterprise-learning-skill 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P5-001 | P0 | A30 | SkillSystem 只读加载 | `done` | `internal/skill` | 本地扫描、去重、排序、每任务最多 3 个技能 |
| AG-P5-002 | P0 | A31 | SkillSecurityScanner | `done` | `internal/skill` | 注入、SSRF、路径穿越规则可测 |
| AG-P5-003 | P1 | A32 | SkillInstaller | `done` | `internal/skill` | registry manifest、SHA-256、磁盘/DB 同步 |
| AG-P5-004 | P0 | A43 | CapabilityMemoryStore | `done` | `internal/memory` | Gene/Capsule 排名、冻结、注入上限 |
| AG-P5-005 | P0 | A50 | SignalExtractor | `done` | `internal/evolution` | failure/success/hardloop/feedback 信号提取 |
| AG-P5-006 | P0 | A51/A52/A53/A54 | GEP 学习进化闭环 | `done` | `internal/evolution` | scan→signal→intent→mutate→validate→solidify，写 X02 |
| AG-P5-007 | P1 | A64 | EvolutionEvaluator | `done` | `internal/quality` | 14 天退化检测，广播 DegradationAlert |

## 下一批建议执行

| 顺序 | 任务 | 原因 |
|------|------|------|
| 1 | - | 当前无 P24 未完成项 |

## P6 integration 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P6-001 | P0 | A00/A30/A43 | CognitiveCore 注入技能与能力记忆 | `done` | `internal/cognitive` | 上下文包含安全技能、Gene/Capsule，draft/frozen 不注入 |
| AG-P6-002 | P0 | A01/A02/A50 | Runtime/Coordinator 发布学习事件 | `done` | `internal/runtime`, `internal/evolution` | success/failure/hardloop 转为 LearningEvent |
| AG-P6-003 | P0 | A12/A51 | BackgroundScheduler 触发 GEP | `done` | `internal/task`, `internal/evolution` | 周期扫描失败学习事件并运行 GepCycle |
| AG-P6-004 | P0 | Storage/A30/A43/A54 | P5 数据 SQLite 持久化 | `done` | `internal/storage` | skills、scan findings、capability assets、evolution events 可持久化 |
| AG-P6-005 | P1 | A64/A43/A62 | 退化治理动作 | `done` | `internal/quality`, `internal/memory`, `internal/approval` | DegradationAlert 可冻结资产并创建审批 |
| AG-P6-006 | P1 | A70/X00 | wait_event/subworkflow 原语 | `done` | `internal/workflow` | 工作流可等待事件恢复并调用子工作流 |

## P7 production-readiness 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P7-001 | P0 | C70 | MaaS HTTP 适配器 | `done` | `internal/adapter` | `HTTPMaasClient` 实现 `MaasInferenceClient`，支持 JSON POST、Bearer token、非 2xx 错误 |
| AG-P7-002 | P0 | Storage/A10/A70/X00/X02 | 跨进程恢复 | `done` | `internal/storage`, `internal/app`, `internal/workflow` | 重启后可恢复 task、workflow waiting 状态、event、audit |
| AG-P7-003 | P1 | CLI/TUI/C70 | CLI real run mode | `done` | `internal/cli`, `internal/app`, `internal/tui` | `agent run` 支持真实 MaaS URL/API key 和自定义 prompt |
| AG-P7-004 | P1 | Build/Test | 端到端演示脚本 | `done` | `Makefile`, `docs/agents/legion-agent` | 一条命令可跑 demo、workflow、storage smoke test |

## P8 service-readiness 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P8-001 | P0 | Config/C70/Storage/Server | 配置文件加载 | `done` | `internal/config`, `internal/cli`, `internal/app` | 支持 JSON 配置、默认值、环境变量覆盖 |
| AG-P8-002 | P0 | Service/Runtime/A12 | 服务化启动模式 | `done` | `cmd`, `internal/service`, `internal/app` | `agent serve` 可启动并优雅停止 background scheduler |
| AG-P8-003 | P0 | HTTP API/A10/A70 | HTTP API 面 | `done` | `internal/server` | health、task submit/status、workflow waiting 列表可测 |
| AG-P8-004 | P1 | Storage/CLI/Service | 真实持久化运行模式 | `done` | `internal/app`, `internal/storage`, `internal/cli` | CLI/service 可通过配置切换 SQLite event/audit/task state |
| AG-P8-005 | P1 | CI/Release | CI 发布流水线 | `done` | `.github/workflows`, `Makefile` | test/vet/build/smoke 在 CI 中可重复执行 |

## P9 operability-release 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P9-001 | P0 | HTTP/Security/Trace | HTTP 管理令牌与请求追踪 | `done` | `internal/config`, `internal/server`, `internal/cli` | 管理接口 token 保护，health 可配置公开，响应带 `X-Request-ID` |
| AG-P9-002 | P0 | Observability/Logging | 结构化日志与组件字段规范 | `done` | `internal/observability`, `internal/service`, `internal/server`, `internal/app`, `internal/cli` | 日志包含 level/msg/component/request_id/task_id，不输出密钥或 prompt |
| AG-P9-003 | P0 | Observability/Metrics | 内存指标与 `/metrics` 诊断面 | `done` | `internal/observability`, `internal/server`, `internal/app`, `internal/cli` | task/http/model/approval/workflow 指标可 JSON 导出 |
| AG-P9-004 | P0 | Health/Diagnostics | Readiness、诊断快照与敏感信息净化 | `done` | `internal/observability`, `internal/server`, `internal/storage`, `internal/cli` | `/readyz` 反映依赖状态，diagnostics 不泄露 secret/prompt |
| AG-P9-005 | P1 | Storage/Ops | SQLite schema 版本、备份与恢复命令 | `done` | `internal/storage`, `internal/cli`, `docs/agents/legion-agent/storage-ops.md` | 迁移幂等，backup 带 checksum，restore 失败不覆盖目标库 |
| AG-P9-006 | P1 | Release/CLI/CI | 版本命令与发布构建脚本 | `done` | `internal/version`, `internal/cli`, `scripts/release.ps1`, `.github/workflows`, `docs/agents/legion-agent/release.md` | `agent version` 可注入版本，release 脚本生成三平台产物 |
| AG-P9-007 | P1 | Compat/CI | API、配置、Workflow DSL 兼容性门禁 | `done` | `internal/compat`, `.github/workflows`, `docs/agents/legion-agent/ci.md` | 最小 config、HTTP 核心字段、workflow DSL golden 测试通过 |
| AG-P9-008 | P1 | Docs/Ops | 运维 Runbook 与 P9 总验收 | `done` | `docs/agents/legion-agent`, `README.md` | operations/release/storage 文档可指导部署、备份、恢复、回滚 |

## P10 component-parity 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P10-001 | P0 | Registry/Compat | 组件实现对齐清单与 parity gate | `done` | `internal/compat`, `docs/agents/legion-agent` | A00-A70/X00-X05/C70 的实现状态、测试覆盖、降级策略可追踪 |
| AG-P10-002 | P0 | A00/A03/C70 | ContextCompressor 四层策略与 checkpoint | `done` | `internal/cognitive`, `internal/adapter` | trim/protect/summary/checkpoint 可测；C70 缺失时零 LLM 降级 |
| AG-P10-003 | P0 | A20-A23/X04/X05 | ToolRegistry 完整执行流水线 | `done` | `internal/tool`, `internal/port` | descriptor/schema/policy/permission/guardrails/timeout/sanitize 全链路可测 |
| AG-P10-004 | P1 | A30-A32 | Skill lifecycle 强化 | `done` | `internal/skill`, `internal/storage` | registry manifest、quarantine、install audit、disable/enable 可测 |
| AG-P10-005 | P0 | A52-A54/A43/X02 | Gene 蒸馏、固化与 sealed evolution log | `done` | `internal/evolution`, `internal/memory`, `internal/storage` | Gene 六元组、alpha 非空、版本寻址、immutable seal 可测 |
| AG-P10-006 | P1 | A60-A64/X00/X02 | 质量趋势与信任治理历史化 | `done` | `internal/quality`, `internal/storage`, `internal/observability` | eval/trust/degradation 历史可查，metrics/diagnostics 可见 |
| AG-P10-007 | P0 | A70/A10/A62/X00 | Workflow 高级原语与服务 API | `done` | `internal/workflow`, `internal/server`, `internal/storage` | loop/join/quorum/timeout，submit/resume/list API 可测 |
| AG-P10-008 | P1 | CI/Docs | P10 总验收与文档同步 | `done` | `.github/workflows`, `docs/agents/legion-agent`, `docs/plans/03-agent` | compat/parity/test/vet/build/smoke/release 全部通过 |

## P11 platform-integration 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P11-001 | P0 | API/X05/Compat | OpenAPI 3.1 契约与兼容性门禁 | `done` | `internal/server`, `internal/compat`, `docs/agents/legion-agent` | `/openapi.json` 可用，golden 防破坏，schema 不暴露 secret/prompt |
| AG-P11-002 | P0 | X00/A01/A02/A60-A64/A70 | EventBus 与 SSE 平台事件流 | `done` | `internal/observability`, `internal/server` | task/workflow/quality 事件可订阅、可过滤、断开可清理 |
| AG-P11-003 | P0 | A02/A10/A70/X02 | Tenant/company 安全边界 | `done` | `internal/security`, `internal/server`, `internal/storage` | 跨 company 访问 task/workflow 被拒绝并审计；quality/audit 查询 API 待新增时复用同一边界 |
| AG-P11-004 | P1 | Observability/X00/X02 | Prometheus text exporter 与外部观测字段稳定化 | `done` | `internal/observability`, `internal/server`, `docs/agents/legion-agent` | `/metrics?format=prometheus` 输出稳定，字段无 prompt/secret |
| AG-P11-005 | P1 | Storage/X02/A60-A64 | 数据保留、质量历史归档与导出 | `done` | `internal/storage`, `internal/cli`, `docs/agents/legion-agent` | retention dry-run/apply 可测，归档摘要写 audit |
| AG-P11-006 | P1 | CI/Docs | P11 总验收与文档索引同步 | `done` | `.github/workflows`, `docs/agents/legion-agent`, `docs/plans/03-agent` | openapi/event/security/retention/test/vet/build/smoke 全部通过 |

## P12 enterprise-governance-ecosystem 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P12-001 | P0 | Security/X02/A60-A64 | RBAC 与 audit/quality 查询边界 | `done` | `internal/security`, `internal/server`, `internal/storage` | audit/quality 查询按 role 控制，拒绝写 X02；company schema 扩展前保持 task/workflow company 边界 |
| AG-P12-002 | P0 | A30-A32/X03/X04/X02 | 远端 Skill registry 同步 | `done` | `internal/skill`, `internal/config`, `internal/cli`, `docs/agents/legion-agent` | registry sync 走 hash、扫描、quarantine、审计 |
| AG-P12-003 | P1 | C70/A01/A60 | MaaS 多模型 profile 与路由 | `done` | `internal/config`, `internal/adapter`, `internal/cli` | CLI/config 可选择 profile，runtime 仍只依赖 C70 |
| AG-P12-004 | P1 | Observability/X00/X05 | Trace recorder 与 `/debug/traces` | `done` | `internal/observability`, `internal/server` | trace snapshot 可导出，敏感字段被净化 |
| AG-P12-005 | P1 | API/Compat/X05 | OpenAPI 错误矩阵与客户端兼容样例 | `done` | `internal/server`, `internal/compat`, `docs/agents/legion-agent` | 错误响应 schema、HTTP status、示例请求响应有 golden 门禁 |
| AG-P12-006 | P1 | CI/Docs | P12 总验收与文档索引同步 | `done` | `.github/workflows`, `docs/agents/legion-agent`, `docs/plans/03-agent` | P12 专项、test/vet/build/smoke/release 全部通过 |

## P13 runtime-context-memory 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P13-001 | P0 | Config | context_files/workspace 配置 | `done` | `internal/config` | config 可加载路径、开关、大小限制 |
| AG-P13-002 | P0 | ContextFiles/X05/X04 | 安全加载器 | `done` | `internal/contextfiles` | 只读允许路径，扫描注入，截断大文件 |
| AG-P13-003 | P0 | A00 | CognitiveCore 注入上下文文件 | `done` | `internal/cognitive` | prompt 包含身份、项目规则、工具策略、用户/记忆快照 |
| AG-P13-004 | P0 | A01/C70 | Runtime/App/CLI 接入 | `done` | `internal/runtime`, `internal/app`, `internal/cli` | `agent run` 真实传递 context prefix |
| AG-P13-005 | P1 | Workspace/A40 | docs/memory 目录约束和模板 | `done` | `docs`, `memory`, `configs/persona` | docs_root/memory_root 配置、README、MEMORY.md 模板齐全 |
| AG-P13-006 | P1 | Docs/CI | 文档、任务表、专项验证 | `done` | `.github/workflows`, `docs` | 专项测试、文档索引和 CI 已同步 |

## P14 runtime-cognitive-integration 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P14-001 | P0 | A01/A00 | Runtime 增加 ContextBuilder 并调用 BuildContext | `done` | `internal/runtime` | MaaS prompt 来自 CognitiveCore |
| AG-P14-002 | P0 | App/P13 | App 将 context files block 注入 CognitiveCore | `done` | `internal/app`, `internal/cli` | CLI context files 仍进入 MaaS prompt |
| AG-P14-003 | P1 | Compat/Docs | parity 与任务表同步 | `done` | `internal/compat`, `docs/plans/03-agent` | A00/A01 纳入 component parity |

## P15 runtime-interrupt 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P15-001 | P0 | A01 | Runtime 增加 `Interrupt()` 和 `ErrInterrupted` | `done` | `internal/runtime` | 中断后返回可匹配错误 |
| AG-P15-002 | P0 | A01/X00/A50 | 中断时发布轻量学习事件 | `done` | `internal/runtime`, `internal/evolution` | EventBus 包含 `reason=interrupted lightweight=true` |
| AG-P15-003 | P1 | Docs/Compat | 任务表同步 | `done` | `docs/plans/03-agent` | P15 标记完成 |

## P16 runtime-default-ports 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P16-001 | P0 | A01/X00/X02 | Runtime 缺失 EventBus/AuditLog 时注入 Noop | `done` | `internal/runtime` | 仅配置 MaaS 也能完成任务 |
| AG-P16-002 | P0 | A01/C70 | Runtime 缺失 MaaS 时返回 `ErrMaasUnavailable` | `done` | `internal/runtime` | 错误可通过 `errors.Is` 匹配 |
| AG-P16-003 | P1 | Docs/Compat | 计划与任务表同步 | `done` | `docs/plans/03-agent` | P16 标记完成 |

## P17 interactive-tui 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P17-001 | P0 | TUI | 新增 InteractiveModel | `done` | `internal/tui` | 输入 prompt 后运行任务，界面不自动退出 |
| AG-P17-002 | P0 | CLI/P13 | 新增 `agent tui` 命令并加载五类上下文文件 | `done` | `internal/cli` | context prefix 包含 AGENTS/MEMORY/SOUL/TOOLS/USER |
| AG-P17-003 | P1 | Docs/Validation | 计划、任务表和验证同步 | `done` | `docs/plans/03-agent` | 测试、vet、build 通过 |

## P18 tui-visual-polish 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P18-001 | P0 | CLI/TUI | `agent tui` 启用 alternate screen | `done` | `internal/cli` | 进入 TUI 时清屏，退出后恢复终端 |
| AG-P18-002 | P0 | TUI | TUI 分区面板与状态栏 | `done` | `internal/tui` | PROMPT、RESULT、EVENT STREAM、AUDIT、STATUS 区域稳定展示 |
| AG-P18-003 | P1 | Docs/Validation | 计划、任务表和验证同步 | `done` | `docs/plans/03-agent` | TUI 布局专项测试通过 |

## P19 tui-workbench-layout 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P19-001 | P0 | TUI/Layout | 增加窗口尺寸状态和全屏工作台布局 | `done` | `internal/tui` | View 包含 header、main、plan、composer、footer 五区 |
| AG-P19-002 | P0 | TUI/Composer | 底部输入区改为固定 Composer 体验 | `done` | `internal/tui` | 空输入显示 `编写任务或使用 /。`，输入时显示当前 prompt |
| AG-P19-003 | P1 | TUI/Result | 会话区展示结果、事件流和错误状态 | `done` | `internal/tui` | 运行完成后左侧区域展示 Result/Event/Audit 摘要 |
| AG-P19-004 | P1 | Tests/Docs | 补充布局专项测试并同步任务表 | `done` | `docs/plans/03-agent` | TUI 专项测试和全量验证通过 |
| AG-P19-005 | P1 | TUI/Command | 默认隐藏 Audit，新增 `/audit` 本地命令 | `done` | `internal/tui` | 普通结果视图不显示 AUDIT，输入 `/audit` 后显示审计动作 |
| AG-P19-006 | P1 | TUI/Command | 默认隐藏 Event，新增 `/event` 本地命令 | `done` | `internal/tui` | 普通结果视图不显示 EVENT，输入 `/event` 后显示事件流 |
| AG-P19-007 | P1 | TUI/Logging | 默认隐藏 Status，运行日志写入文件 | `done` | `internal/tui`, `internal/cli` | TUI 不显示 STATUS running/error，CLI 默认日志追加到 `logs/agent.log` |
| AG-P19-008 | P1 | TUI/Composer | Slash 命令提示与历史导航 | `done` | `internal/tui` | 输入 `/` 显示 `/audit`、`/event`；slash 模式上下方向键选命令，普通模式上下方向键浏览历史输入 |
| AG-P19-009 | P1 | TUI/Result | 格式化 Result 输出 | `done` | `internal/tui` | Result 视图分离 Task 与 Output，保留多行正文 |
| AG-P19-010 | P1 | TUI/Display | 模型显示配置化 | `done` | `internal/tui`, `internal/cli` | Header/Footer 使用选中 MaaS profile 的模型名，不硬编码 DeepSeek 字符串 |
| AG-P19-011 | P1 | TUI/Display | 初始化品牌区居中与颜色统一 | `done` | `internal/tui` | 空态主工作区将 `Legion Agent TUI` 和 `role · model` 居中显示，并使用一致主题色 |
| AG-P19-012 | P1 | TUI/Composer | 运行期间 Composer 工作状态 | `done` | `internal/tui` | 与大模型通讯时显示等待输出状态，并忽略输入编辑/历史/命令选择 |
| AG-P19-013 | P1 | TUI/Result | Markdown 结果格式化 | `done` | `internal/tui` | 编号/无序列表不保留孤立 marker，粗体标记转纯文本，长行按主面板宽度挂起缩进 |
| AG-P19-014 | P1 | TUI/Footer | 运行期间底部进度条 | `done` | `internal/tui` | Footer 显示 `工作中 ...` 和横向进度条，空闲快捷键仅在非运行态显示 |
| AG-P19-015 | P1 | TUI/Main | 对话流交互模式 | `done` | `internal/tui` | 主工作区显示用户 prompt、thinking 状态和带 `●` 前缀的模型回复 |
| AG-P19-016 | P1 | TUI/Footer | 动态工作进度条 | `done` | `internal/tui` | 运行期间 tick 推进进度条游标，任务结束后停止继续 tick |
| AG-P19-017 | P1 | TUI/Config | Prompt 与 thinking 展示配置化 | `done` | `internal/config`, `internal/cli`, `internal/tui`, `internal/port`, `internal/adapter`, `internal/runtime`, `internal/app` | `tui.show_prompt` 与 `tui.show_thinking` 控制主会话区是否显示输入问题和 thinking；thinking 优先使用模型服务返回的公开 reasoning |
| AG-P19-018 | P1 | TUI/Quality | 伪工具调用输出治理 | `done` | `configs/persona/TOOLS.md`, `internal/quality`, `internal/app` | 默认工具策略禁止伪工具文本输出，模型输出中出现 `search_content(...)` 等调用时替换为明确能力边界说明 |
| AG-P19-019 | P0 | A01/A20/C70 | 模型驱动工具调用闭环 | `done` | `internal/port`, `internal/adapter`, `internal/runtime`, `internal/tool`, `internal/app` | MaaS/OpenAI-compatible `tool_calls` 可解析为 `ToolCall`，Runtime 执行 `search_content/read_file` 后二次调用模型生成最终答案 |
| AG-P19-020 | P0 | TUI/A20/X00 | 工具输出不截断与流式展示 | `done` | `internal/tool`, `internal/runtime`, `internal/cli`, `internal/tui`, `configs/persona/TOOLS.md` | 内置 `list_files/read_file/search_content` 不在源头截断输出；Runtime 发布完整 `tool_result`；TUI 通过流式事件总线边运行边展示工具输出 |
| AG-P19-021 | P1 | TUI/Output | 输出区鼠标滚轮滚动 | `done` | `internal/tui`, `internal/cli` | TUI 启用 Bubble Tea mouse cell motion；鼠标滚轮在主输出区内向上/向下滚动长输出，Composer 历史导航不受影响 |

## P20 multi-agent-runtime-routing 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P20-001 | P0 | Scheduler | 保留显式 Task.AgentID | `done` | `internal/task` | workflow task.agent_id 不被默认 coordinator 覆盖 |
| AG-P20-002 | P0 | Config/AgentRegistry | 根 agents 配置与子 agent loader | `done` | `internal/config`, `internal/agentregistry` | 可加载 researcher/writer 子配置，缺失文件报明确错误 |
| AG-P20-003 | P0 | Coordinator/A01 | per-agent runtime resolver | `done` | `internal/runtime` | 已注册 agent_id 使用对应 role/context/model/tool root |
| AG-P20-004 | P0 | CLI/Service/Workflow | serve 注入 workflow + coordinator 闭环 | `done` | `internal/cli`, `internal/service`, `internal/server` | POST workflow 后后台 heartbeat 可执行 task |
| AG-P20-005 | P1 | Docs/Examples | 示例配置与参考文档同步 | `done` | `configs`, `docs/agents`, `docs/plans` | 配置示例、协作参考、计划索引一致 |

## P21 agent-message-bus 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P21-000 | P0 | Docs/Config | `tasks.md` 文件态协作配置与规范 | `done` | `internal/config`, `configs`, `docs/agents/reference` | 已新增 `tasks` 配置块、参考手册和 P21 映射说明 |
| AG-P21-F01 | P0 | Domain/File | TaskLedger append-only event store 与投影服务 | `done` | `internal/taskledger` | 已新增事件 schema、append-only JSONL、锁、幂等 replay、deterministic projection、并发追加和路径边界测试 |
| AG-P21-F02 | P0 | Tool/Runtime | tasks 真实工具 | `done` | `internal/tool`, `internal/runtime`, `internal/app`, `internal/cli` | 已新增 `create_task`、`claim_task`、`update_task`、`append_task_message`、`read_task`、`rebuild_tasks`，并注入 CLI/TUI/serve/子 Agent Runtime |
| AG-P21-F03 | P0 | TUI | tasks 协作命令 | `done` | `internal/tui`, `internal/cli` | 已支持 `/tasks`、`/task <id>`、`/handoff <agent> <task_id> <summary>`，并在 TUI 中显示 TaskLedger 投影 |
| AG-P21-F04 | P1 | TUI/@Agent | `@agent --task` 任务绑定 | `done` | `internal/tui`, `internal/cli` | `@researcher --task TASK-*` / `@writer --task TASK-*` 会注入 TaskLedger 任务详情，并在模型完成后追加 `result.appended`、重建投影 |
| AG-P21-F05 | P1 | Guard/Archive | tasks 并发冲突、膨胀控制与归档 | `done` | `internal/taskledger` | owner claim 冲突在详情投影中显示 actor/owner；`max_index_lines` / `max_task_lines` 超阈值给出归档或拆分提示；done/cancelled 写入 `tasks/archive/` 并移除活跃详情投影 |
| AG-P21-F06 | P1 | Docs/Compat | tasks 并发协作 smoke 与文档同步 | `done` | `docs/agents`, `internal/compat` | 已新增 compat golden smoke，覆盖 researcher/writer/reviewer 并发追加、replay deterministic、消息不丢失、`tasks.md` 不泄漏消息历史，并同步参考文档 |
| AG-P21-001 | P0 | Domain/Storage | AgentMessage 数据模型与 SQLite 持久化 | `done` | `internal/domain`, `internal/storage` | 已新增 AgentMessage/Query/TaskEventFields 映射，SQLite `agent_messages` 可创建、按 recipient/status/task 查询、标记已读，并保留 P21A source_event_id |
| AG-P21-002 | P0 | A20/Runtime | `send_message` / `read_messages` 工具 | `done` | `internal/tool`, `internal/runtime`, `internal/app`, `internal/cli` | 已新增 `send_message` / `read_messages` 工具，支持发送、按条件读取、`mark_read` 标记已读，并注入 CLI/TUI/serve/default runtime/子 Agent runtime |
| AG-P21-003 | P0 | TUI | `/send <agent> <message>` 与 `/inbox` 命令 | `done` | `internal/tui`, `internal/cli` | P21B：TUI 已可查看当前 Agent 未读 inbox，并通过持久化 message store 向目标 Agent 发送消息 |
| AG-P21-004 | P1 | TUI/@Agent | `@agent` 支持消息式 handoff | `done` | `internal/tui`, `internal/cli` | P21B：`@writer --inbox` 可读取目标 Agent 未读消息继续处理，成功后标记已读，失败保持未读 |
| AG-P21-005 | P1 | Workflow | task result 传递到后续 task input/message | `done` | `internal/workflow` | P21C：后续 task input 已可通过 `{{tasks.<task_id>.result}}` 引用 `task_completed` 事件中的前序 task result |
| AG-P21-006 | P1 | API | message 查询/发送 HTTP 接口 | `done` | `internal/server`, `internal/cli`, `internal/compat` | P21B/P21C：`GET/POST /v1/agents/{id}/messages` 可发送、查询和 `mark_read`，OpenAPI golden 已同步 |
| AG-P21-007 | P1 | Docs/Compat | 协作文档与兼容性测试 | `done` | `docs/agents`, `internal/compat` | 已新增 P21 collaboration surface compat golden，文档区分 routing、session、TaskLedger、message bus、workflow handoff、HTTP message API 六类协作 |

## P22 session-context-continuity 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P22-001 | P0 | Domain/Storage | AgentSession 与 ConversationTurn 数据模型和 SQLite 持久化 | `done` | `internal/domain`, `internal/storage` | 可创建 session、追加 turn、按 session 查询 turns |
| AG-P22-002 | P0 | TUI/CLI | TUI 启动创建/恢复当前 session | `done` | `internal/tui`, `internal/cli` | TUI footer/header 显示 session，重启可恢复最近 session |
| AG-P22-003 | P0 | Runtime/A00 | 最近 N 轮 turn 注入 CognitiveCore prompt | `done` | `internal/cognitive`, `internal/runtime`, `internal/cli` | 第二轮请求包含上一轮用户问题与 Agent 回复摘要 |
| AG-P22-004 | P1 | TUI Command | `/new`、`/sessions`、`/switch`、`/clear-session` | `done` | `internal/tui`, `internal/cli` | 用户可创建、切换、清空会话 |
| AG-P22-005 | P1 | Multi-Agent | session turn 记录 agent_id 与 model profile | `done` | `internal/domain`, `internal/storage`, `internal/cli` | `@researcher`、`@writer` 在同一 session 中可追踪各自输出 |
| AG-P22-006 | P1 | Context Policy | 上下文窗口、截断和敏感信息过滤 | `done` | `internal/cognitive`, `internal/config`, `internal/quality` | 最近 N 轮可配置，超长内容被安全截断 |
| AG-P22-007 | P1 | API/Docs | HTTP session 查询接口与文档同步 | `done` | `internal/server`, `docs/agents`, `docs/plans` | 可查询 session/turn，配置文档说明 session 行为 |

## P23 session-context-cache 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P23-001 | P0 | Config/Plan | 增加 session cache 配置与计划索引 | `done` | `internal/config`, `docs/plans` | `session.cache_enabled`、`session.cache_max_entries` 可加载 |
| AG-P23-002 | P0 | Cache | 实现内存 SessionContextCache | `done` | `internal/sessioncache` | 支持 get/put/invalidate/stats，返回副本且线程安全 |
| AG-P23-003 | P0 | TUI/CLI | `RecentTurns` 接入 cache 和失效策略 | `done` | `internal/cli` | 连续读取命中 cache，追加 turn 后失效 |
| AG-P23-004 | P1 | Docs | 更新 session/cache 使用说明 | `done` | `docs/agents/reference` | 手册说明 cache 与 memory 的边界 |
| AG-P23-005 | P1 | Verification | 兼容性与总验收 | `done` | `internal/compat`, `docs/plans` | `go test ./...`、`go vet ./...`、`go build -o NUL ./cmd` 通过 |

## P24 role-scoped-skills 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P24-001 | P0 | AgentRegistry/A01/A00/A30 | 子 Agent Runtime 按角色 skills root 注入 SkillSystem | `done` | `internal/runtime`, `internal/skill` | `@researcher` 只挂载 researcher skills；未配置时继承根 `skills.install_root` |
| AG-P24-002 | P0 | CLI/TUI/A32 | skill 管理命令支持指定 agent | `done` | `internal/cli`, `internal/tui` | `agent skill sync --agent writer` 和 TUI `/skill ... --agent writer` 写入 writer 的 install root |
| AG-P24-003 | P1 | Config/Docs/Tests | 示例配置、参考手册与回归测试同步 | `done` | `configs`, `docs/agents/reference`, `docs/plans` | 文档说明全局/角色 skills 继承规则，测试覆盖角色隔离和回退 |
