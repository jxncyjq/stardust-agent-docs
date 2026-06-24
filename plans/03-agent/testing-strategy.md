---
id: "plans-agent-testing-strategy-001"
title: "Agent 测试策略"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "testing", "golang"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# Agent 测试策略

## 测试层级

| 层级 | 范围 | 示例 |
|------|------|------|
| 单元测试 | 单包逻辑 | TaskLock、ExecutionPolicy、ApprovalService |
| 契约测试 | 端口接口 | MaasInferenceClient fake/recording、AuditLog |
| 集成测试 | 内存装配 | Runtime + Scheduler + Fake MaaS |
| 回归测试 | MVP 闭环 | fake task 从 pending 到 done |

## 第一批必测用例

| 用例 | 风险 |
|------|------|
| 同一任务并发锁定只有一个成功 | 重复执行 |
| AgentRuntime 只通过 C70 调模型 | 绕过 MaaS 边界 |
| 工具调用被 PermissionPolicy 拦截 | 权限越界 |
| HardLoop 创建 suspended task 和审批工单 | 异常不可治理 |
| Aegis 拒绝后任务不进入 done | 质量门控失效 |
| AuditLog 记录 task/model/tool/approval | 不可追溯 |
| ContextCompressor 超阈值触发压缩或 Noop 降级 | 上下文失控 |
| MemoryProvider 只通过 X01 检索向量 | 绕过公共嵌入端口 |
| SkillSecurityScanner 拦截危险技能 | 技能注入风险 |
| WorkflowEngine approval 节点能暂停和恢复 | 多 Agent 流程不可治理 |

## 按阶段测试门槛

| 阶段 | 必须通过 |
|------|----------|
| P2 | `runtime`、`task`、`tool`、`approval`、`quality` 单元和内存集成测试 |
| P3 | `memory`、`background scheduler`、`event replay` 测试 |
| P4 | `trust score`、`approval full`、`workflow` 状态机测试 |
| P5 | `skill scanner`、`gep cycle`、`evolution evaluator` 测试 |

## 命令

```text
go test ./...
go test ./internal/task -run TestTaskLock
go test ./internal/runtime -run TestRuntime
go build ./cmd
```

## 约定

- 测试优先使用 table-driven tests。
- fake 依赖必须 deterministic。
- 不依赖真实网络、真实 MaaS、真实 Know。
- 并发测试要使用 context timeout，避免挂死。
