---
id: "plans-agent-index-000"
title: "Agent 引擎计划索引"
type: "index"
category: "plans/agent"
tags: ["plan", "agent-engine"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-06-01"
status: "active"
---

# Agent 引擎计划索引

## 当前执行焦点

当前第一目标为 **持续优化 Agent**。

- 继续以 `legion/legionAgent` 为主线，优先完善 TUI、配置、上下文文件、记忆目录、运行稳定性和本地开发体验。
- MaaS 网关服务端暂不进入当前编码批次；Agent 侧保留 C70 HTTP 客户端和 MaaS profile 作为过渡接入能力。
- 后续如需内部 MaaS 协议，可让 Agent 指向独立 MaaS 网关的 `/v1/inference/generate`；该网关服务端实施计划见 [../02-maas/index.md](../02-maas/index.md)。

| 文档 | 说明 |
|------|------|
| [task-breakdown.md](./task-breakdown.md) | Agent 开发任务详情表，按阶段跟踪状态、代码位置和验收 |
| [component-implementation-roadmap.md](./component-implementation-roadmap.md) | A00-A70 组件实施矩阵、profile 分期和验收路线 |
| [go-repository-plan.md](./go-repository-plan.md) | Agent 独立 Go 仓库结构、入口、包边界和工程约定 |
| [cli-tui-plan.md](./cli-tui-plan.md) | Agent CLI 命令与 Bubble Tea 终端 UI 展示计划 |
| [mvp-contracts.md](./mvp-contracts.md) | Agent MVP 核心接口、外部端口和领域对象契约 |
| [data-model-plan.md](./data-model-plan.md) | Company/Agent-lite、Task、Approval、Audit 等数据模型计划 |
| [runtime-mvp-plan.md](./runtime-mvp-plan.md) | Agent 最小运行闭环计划 |
| [sprint-01-scaffold.md](./sprint-01-scaffold.md) | 第一轮 scaffold 与可运行假闭环任务清单 |
| [quality-safety-governance-plan.md](./quality-safety-governance-plan.md) | A60-A64 质量、安全、审批、信任治理实施计划 |
| [skill-system-plan.md](./skill-system-plan.md) | A30-A32 技能系统、扫描和安装实施计划 |
| [workflow-engine-plan.md](./workflow-engine-plan.md) | A70 工作流 DSL 与多 Agent 编排实施计划 |
| [testing-strategy.md](./testing-strategy.md) | Agent 独立仓库测试策略 |
| [learning-quality-plan.md](./learning-quality-plan.md) | 记忆、学习进化与质量治理计划 |
| [p9-operability-release-plan.md](./p9-operability-release-plan.md) | P9 运维化、观测性、安全边界、发布与兼容性计划 |
| [p10-component-parity-plan.md](./p10-component-parity-plan.md) | P10 按 Agent 组件规范补齐高级行为与契约门禁计划 |
| [p11-platform-integration-plan.md](./p11-platform-integration-plan.md) | P11 平台集成就绪计划，补齐 OpenAPI、事件流、租户边界、观测出口和数据保留 |
| [p12-enterprise-governance-ecosystem-plan.md](./p12-enterprise-governance-ecosystem-plan.md) | P12 企业治理与外部生态硬化计划，补齐 RBAC、Skill registry、模型 profile、trace 和错误契约 |
| [p13-runtime-context-memory-plan.md](./p13-runtime-context-memory-plan.md) | P13 运行时上下文文件与记忆目录计划，补齐 SOUL、AGENTS、TOOLS、USER、MEMORY 和 docs/memory 工作空间 |
| [p14-runtime-cognitive-integration-plan.md](./p14-runtime-cognitive-integration-plan.md) | P14 Runtime 与 CognitiveCore 集成计划，让 A01 正式通过 A00 组装 MaaS prompt |
| [p15-runtime-interrupt-plan.md](./p15-runtime-interrupt-plan.md) | P15 Runtime 中断控制计划，补齐 A01 Interrupt 控制面和轻量学习事件 |
| [p16-runtime-default-ports-plan.md](./p16-runtime-default-ports-plan.md) | P16 Runtime 默认端口与缺失依赖保护计划，补齐 Noop Event/Audit 和 MaaS 缺失错误 |
| [p17-interactive-tui-plan.md](./p17-interactive-tui-plan.md) | P17 交互式 TUI 计划，新增 `agent tui` 输入 prompt、运行任务并加载五类上下文文件 |
| [p18-tui-visual-polish-plan.md](./p18-tui-visual-polish-plan.md) | P18 TUI 视觉与交互增强计划，启用全屏 alternate screen 和 Bubble Tea 分区面板 |
| [p19-tui-workbench-layout-plan.md](./p19-tui-workbench-layout-plan.md) | P19 TUI 工作台布局计划，参考 DeepSeek-TUI 构建 header/main/plan/composer/footer 全屏界面 |
| [p20-multi-agent-runtime-routing-plan.md](./p20-multi-agent-runtime-routing-plan.md) | P20 多 Agent Runtime 路由计划，补齐 AgentRegistry、Scheduler AgentID 保留、Coordinator per-agent Runtime 与 serve 工作流闭环 |
| [p21-agent-message-bus-plan.md](./p21-agent-message-bus-plan.md) | P21 Agent 通讯计划：P21A TaskLedger、P21B inbox/outbox 基础层、TUI `/send` `/inbox`、`@agent --inbox`、workflow result handoff、message HTTP API 和协作文档/兼容性测试已完成 |
| [p22-session-context-continuity-plan.md](./p22-session-context-continuity-plan.md) | P22 Session Context Continuity 会话上下文连续性计划，补齐 session/turn 持久化、最近 N 轮注入、TUI session 命令和多 Agent 会话线程 |
| [p23-session-context-cache-plan.md](./p23-session-context-cache-plan.md) | P23 Session Context Cache 计划，在 P22 session continuity 之上补齐上下文窗口 cache、失效策略和 cache 观测 |
| [p24-role-scoped-skills-plan.md](./p24-role-scoped-skills-plan.md) | P24 Role Scoped Skills 计划，让多 Agent 分角色后 skills 安装、加载和 prompt 注入跟随角色配置 |

## 阶段主线

| 阶段 | 主计划 | 核心组件 |
|------|--------|----------|
| P2 minimal | [runtime-mvp-plan.md](./runtime-mvp-plan.md) · [sprint-01-scaffold.md](./sprint-01-scaffold.md) | A00/A01/A02/A03/A10/A11/A20-A23/A60/A62-lite/A63-lite |
| P3 standard-memory | [component-implementation-roadmap.md](./component-implementation-roadmap.md) · [learning-quality-plan.md](./learning-quality-plan.md) | A12/A40/A41/A42/X01/X02 |
| P4 governance-workflow | [quality-safety-governance-plan.md](./quality-safety-governance-plan.md) · [workflow-engine-plan.md](./workflow-engine-plan.md) | A61/A62-full/A63-full/A70/X05 |
| P5 enterprise-learning-skill | [skill-system-plan.md](./skill-system-plan.md) · [learning-quality-plan.md](./learning-quality-plan.md) | A30/A31/A32/A43/A50-A54/A64 |
| P6 integration | [task-breakdown.md](./task-breakdown.md) | A00/A01/A02/A12/A30/A43/A50-A54/A62/A64/A70/X00 |
| P7 production-readiness | [task-breakdown.md](./task-breakdown.md) | C70/Storage/CLI/TUI/X00/X02 |
| P8 service-readiness | [task-breakdown.md](./task-breakdown.md) | Config/Service/HTTP API/Storage/CI |
| P9 operability-release | [p9-operability-release-plan.md](./p9-operability-release-plan.md) · [task-breakdown.md](./task-breakdown.md) | HTTP Auth/Observability/Diagnostics/Storage Ops/Release/Compat |
| P10 component-parity | [p10-component-parity-plan.md](./p10-component-parity-plan.md) · [task-breakdown.md](./task-breakdown.md) | A03/A20-A23/A30-A32/A52-A54/A60-A64/A70/Compat |
| P11 platform-integration | [p11-platform-integration-plan.md](./p11-platform-integration-plan.md) · [task-breakdown.md](./task-breakdown.md) | OpenAPI/X00 EventBus/SSE/Tenant/Authz/Prometheus/Retention |
| P12 enterprise-governance-ecosystem | [p12-enterprise-governance-ecosystem-plan.md](./p12-enterprise-governance-ecosystem-plan.md) · [task-breakdown.md](./task-breakdown.md) | RBAC/Audit/Quality/A30-A32/C70/Trace/OpenAPI Errors |
| P13 runtime-context-memory | [p13-runtime-context-memory-plan.md](./p13-runtime-context-memory-plan.md) · [task-breakdown.md](./task-breakdown.md) | A00/A20-A23/A40-A43/X04/X05/ContextFiles |
| P14 runtime-cognitive-integration | [p14-runtime-cognitive-integration-plan.md](./p14-runtime-cognitive-integration-plan.md) · [task-breakdown.md](./task-breakdown.md) | A00/A01/C70 |
| P15 runtime-interrupt | [p15-runtime-interrupt-plan.md](./p15-runtime-interrupt-plan.md) · [task-breakdown.md](./task-breakdown.md) | A01/X00/A50 |
| P16 runtime-default-ports | [p16-runtime-default-ports-plan.md](./p16-runtime-default-ports-plan.md) · [task-breakdown.md](./task-breakdown.md) | A01/X00/X02/C70 |
| P17 interactive-tui | [p17-interactive-tui-plan.md](./p17-interactive-tui-plan.md) · [task-breakdown.md](./task-breakdown.md) | CLI/TUI/A01/P13 |
| P18 tui-visual-polish | [p18-tui-visual-polish-plan.md](./p18-tui-visual-polish-plan.md) · [task-breakdown.md](./task-breakdown.md) | CLI/TUI/Bubble Tea |
| P19 tui-workbench-layout | [p19-tui-workbench-layout-plan.md](./p19-tui-workbench-layout-plan.md) · [task-breakdown.md](./task-breakdown.md) | CLI/TUI/DeepSeek-TUI |
| P20 multi-agent-runtime-routing | [p20-multi-agent-runtime-routing-plan.md](./p20-multi-agent-runtime-routing-plan.md) · [task-breakdown.md](./task-breakdown.md) | Workflow/A01/A00/C70/Config/Service |
| P21 agent-message-bus | [p21-agent-message-bus-plan.md](./p21-agent-message-bus-plan.md) · [task-breakdown.md](./task-breakdown.md) | TaskLedger/AgentMessage/Storage/A20/TUI/Workflow/API |
| P22 session-context-continuity | [p22-session-context-continuity-plan.md](./p22-session-context-continuity-plan.md) · [task-breakdown.md](./task-breakdown.md) | AgentSession/ConversationTurn/Storage/A00/TUI/CLI/API |
| P23 session-context-cache | [p23-session-context-cache-plan.md](./p23-session-context-cache-plan.md) · [task-breakdown.md](./task-breakdown.md) | SessionContextCache/TUI/CLI/Config/Docs |
| P24 role-scoped-skills | [p24-role-scoped-skills-plan.md](./p24-role-scoped-skills-plan.md) · [task-breakdown.md](./task-breakdown.md) | A30-A32/AgentRegistry/A00/A01/TUI/CLI |

## 当前状态

| 阶段 | 状态 | 说明 |
|------|------|------|
| P2-P8 | `done` | Agent MVP、记忆、治理、学习、集成、生产准备、服务化入口均已完成 |
| P9 | `done` | 运维化、安全边界、可观测、诊断、SQLite 运维、发布、兼容性和 runbook 已完成 |
| P10 | `done` | 组件高级行为、parity gate、A60-A64 历史化、A70 高级工作流和 P10 总验收已完成 |
| P11 | `done` | 平台集成就绪：OpenAPI 契约、事件订阅、租户边界、外部观测和数据保留已完成 |
| P12 | `done` | 企业治理与外部生态硬化：RBAC 查询边界、远端 Skill registry、模型 profile、trace 出口、错误契约、CI 门禁与总验收已完成 |
| P13 | `done` | 运行时上下文文件与记忆目录：SOUL、AGENTS、TOOLS、USER、MEMORY 加载、docs/memory 工作空间边界和 CLI 开关已完成 |
| P14 | `done` | Runtime 与 CognitiveCore 集成：A01 已通过 A00 组装 prompt 后调用 C70 |
| P15 | `done` | Runtime 中断控制：Interrupt 可阻断 MaaS 调用并发布 interrupted 轻量学习事件 |
| P16 | `done` | Runtime 默认端口：缺失 EventBus/AuditLog 自动 Noop，缺失 MaaS 返回明确错误 |
| P17 | `done` | 交互式 TUI：新增 `agent tui`，支持输入 prompt、运行任务、停留界面并加载五类上下文文件 |
| P18 | `done` | TUI 视觉与交互增强：进入 TUI 使用全屏 alternate screen，界面改为 Bubble Tea 风格分区面板和状态栏 |
| P19 | `done` | TUI 工作台布局：参考 DeepSeek-TUI 截图构建顶部状态、主工作区、右侧 Plan、底部 Composer 和状态条，并支持 prompt/thinking 展示配置 |
| P20 | `done` | 多 Agent Runtime 路由：`task.agent_id` 可路由到不同 SOUL/MEMORY/model profile/workspace |
| P21 | `done` | Agent Message Bus 与 Agent 通讯：P21A `tasks.md`、SQLite inbox/outbox、send/read message 工具、TUI `/send` `/inbox`、`@agent --inbox`、workflow result handoff、message HTTP API 和协作文档/兼容性测试已完成 |
| P22 | `done` | Session Context Continuity：已补齐 `session_id`、conversation turn 持久化、最近 N 轮上下文注入、TUI 会话命令和 HTTP session 查询接口 |
| P23 | `done` | Session Context Cache：已补齐 session 最近上下文窗口 cache、失效策略和 cache stats |
| P24 | `done` | Role Scoped Skills：主 Agent 使用全局 skills，子 Agent 使用各自 `skills.install_root`，CLI/TUI 管理入口已补齐 |
