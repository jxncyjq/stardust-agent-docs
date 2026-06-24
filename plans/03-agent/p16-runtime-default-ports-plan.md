---
id: "plans-agent-p16-runtime-default-ports-001"
title: "Agent P16 Runtime 默认端口与缺失依赖保护计划"
type: "implementation-plan"
category: "plans/agent"
tags: ["plan", "agent", "p16", "runtime", "noop", "ports"]
version: "0.1.0"
created: "2026-05-17"
updated: "2026-05-17"
status: "done"
related_docs:
  - path: "./task-breakdown.md"
    relation: "updates"
  - path: "../../design/architecture/agent_components/agent-component-registry.md"
    relation: "implements"
---

# Agent P16 Runtime Default Ports Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** 让 `AgentRuntime` 对可选端口使用 Noop 降级，并在必需 MaaS 端口缺失时返回可匹配错误，避免运行期 panic。

**Architecture:** `internal/runtime` 内部提供私有 `noopEventBus` 和 `noopAuditLog`，`NewRuntime` 在 `Events` 或 `Audit` 缺失时自动注入。`RunTask` 在调用 MaaS 前检查 `maas == nil`，返回 `ErrMaasUnavailable`。

**Tech Stack:** Go 1.26.0、现有 `port` 接口、sentinel error、TDD。

---

## 任务清单

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 验收 |
|---------|--------|------|------|------|------|
| AG-P16-001 | P0 | A01/X00/X02 | Runtime 缺失 EventBus/AuditLog 时注入 Noop | `done` | 仅配置 MaaS 也能完成任务 |
| AG-P16-002 | P0 | A01/C70 | Runtime 缺失 MaaS 时返回 `ErrMaasUnavailable` | `done` | 错误可通过 `errors.Is` 匹配 |
| AG-P16-003 | P1 | Docs/Compat | 计划与任务表同步 | `done` | P16 标记完成 |

## 验证

```powershell
go test ./internal/runtime -run "TestRuntimeUsesNoopPortsWhenAuditAndEventsMissing|TestRuntimeMissingMaasReturnsErrMaasUnavailable" -count=1
go test ./...
go vet ./...
go build -o NUL ./cmd
```
