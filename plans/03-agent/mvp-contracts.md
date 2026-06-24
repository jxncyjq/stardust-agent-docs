---
id: "plans-agent-mvp-contracts-001"
title: "Agent MVP 契约计划"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "contracts", "ports"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# Agent MVP 契约计划

## 目标

先定义 Agent MVP 的核心接口和领域对象，确保后续编码围绕一条可运行闭环推进。

## 外部端口

| 端口 | 说明 | MVP 实现 |
|------|------|----------|
| `MaasInferenceClient` | C70 推理入口 | fake + recording |
| `KnowledgeClient` | K61-lite 查询与 K50-lite 提交 | fake |
| `EventBus` | X00 事件发布订阅 | in-memory |
| `AuditLog` | X02 审计写入 | in-memory / file |
| `PathGuard` | X04 路径校验 | workspace-only |
| `OutputSanitizer` | X05 输出净化 | noop + deterministic sanitizer |
| `EmbeddingProvider` | X01 向量嵌入 | P3 fake + deterministic vector |

## 核心服务接口

| 接口 | 组件 | 说明 |
|------|------|------|
| `AgentRuntime.RunTask` | A01 | 执行单个已锁定任务 |
| `TaskScheduler.Next` | A10 | 获取待执行任务 |
| `TaskLock.TryLock` | A11 | 原子锁定任务 |
| `ToolRegistry.Execute` | A20 | 执行工具调用 |
| `AegisReviewer.Review` | A60 | 质量审核 |
| `ApprovalService.OpenTicket` | A62-lite | 创建最小审批工单 |
| `CognitiveCore.BuildContext` | A00 | 组装任务上下文 |
| `ContextCompressor.Compress` | A03 | 超阈值上下文压缩或 Noop 降级 |
| `ExecutionPolicy.Decide` | A21 | 判断工具调用是否允许、拒绝或转审批 |
| `PermissionEnforcer.Check` | A22 | 批量权限检查 |
| `ToolGuardrails.Before/After` | A23 | 工具调用前后风控钩子 |
| `MemoryProvider.Prefetch` | A40 | P3 起为任务加载情景记忆 |
| `WorkflowEngine.Start` | A70 | P4 起启动工作流实例 |

## 领域对象

| 对象 | 必填字段 |
|------|----------|
| `Agent` | id、company_id、role、status、model_policy、permission_policy |
| `Task` | id、company_id、agent_id、status、input、max_iterations、created_at |
| `TaskRun` | id、task_id、agent_id、started_at、ended_at、result |
| `ToolCall` | id、name、arguments、risk_level |
| `ToolResult` | call_id、success、output、error |
| `ApprovalTicket` | id、type、subject_id、status、decision |
| `AuditEvent` | id、request_id、subject_type、subject_id、action、hash |
| `MemoryEntry` | id、company_id、agent_id、kind、content、embedding_ref、created_at |
| `WorkflowRun` | id、workflow_id、company_id、status、current_nodes、created_at |
| `Skill` | id、name、source、version、hash、risk_level、status |

## 契约验收

- 所有端口都有 fake 实现。
- 所有核心服务接口都有最小单元测试。
- Runtime 测试能用 fake 端口跑完整任务。
- 审计事件可按 request_id 关联任务、模型调用、工具调用和审批。
- P2 代码中不得直接出现具体 MaaS provider SDK 调用。
- P3 以后记忆检索必须通过 `EmbeddingProvider` 端口，不能绑定具体向量数据库。
