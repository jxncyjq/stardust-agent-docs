---
id: "plans-agent-p15-runtime-interrupt-001"
title: "Agent P15 Runtime 中断控制计划"
type: "implementation-plan"
category: "plans/agent"
tags: ["plan", "agent", "p15", "runtime", "interrupt", "control"]
version: "0.1.0"
created: "2026-05-17"
updated: "2026-05-17"
status: "done"
related_docs:
  - path: "./task-breakdown.md"
    relation: "updates"
  - path: "../../design/architecture/agent_components/agent-runtime-spec.md"
    relation: "implements"
---

# Agent P15 Runtime Interrupt Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** 为 `AgentRuntime` 补齐可并发调用的 `Interrupt()` 控制面，让外部取消能在 MaaS 调用前被识别并形成轻量学习事件。

**Architecture:** Runtime 内部使用 `atomic.Bool` 记录中断标志，`Interrupt()` 只设置标志，不阻塞、不启动 goroutine。`RunTask` 在调用 MaaS 前检查标志，若已中断则发布 `FailureReasonInterrupted` 的轻量学习事件并返回 `ErrInterrupted`。

**Tech Stack:** Go 1.26.0、`sync/atomic`、现有 `runtime`、`evolution`、`adapter` 测试替身。

---

## 任务清单

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 验收 |
|---------|--------|------|------|------|------|
| AG-P15-001 | P0 | A01 | Runtime 增加 `Interrupt()` 和 `ErrInterrupted` | `done` | 中断后返回可 `errors.Is` 匹配的错误 |
| AG-P15-002 | P0 | A01/X00/A50 | 中断时发布轻量学习事件 | `done` | EventBus 包含 `reason=interrupted lightweight=true` |
| AG-P15-003 | P1 | Docs/Compat | 任务表同步 | `done` | P15 标记完成 |

## 验证

```powershell
go test ./internal/runtime -run TestRuntimeInterruptStopsBeforeInference -count=1
go test ./internal/runtime ./internal/app ./internal/cli -count=1
go test ./...
go vet ./...
go build -o NUL ./cmd
```
