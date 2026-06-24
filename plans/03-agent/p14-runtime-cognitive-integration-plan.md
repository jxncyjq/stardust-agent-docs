---
id: "plans-agent-p14-runtime-cognitive-integration-001"
title: "Agent P14 Runtime 与 CognitiveCore 集成计划"
type: "implementation-plan"
category: "plans/agent"
tags: ["plan", "agent", "p14", "runtime", "cognitive-core", "context"]
version: "0.1.0"
created: "2026-05-17"
updated: "2026-05-17"
status: "done"
related_docs:
  - path: "./task-breakdown.md"
    relation: "updates"
  - path: "../../design/architecture/agent_components/agent-runtime-spec.md"
    relation: "implements"
  - path: "../../design/architecture/agent_components/cognitive-core-spec.md"
    relation: "implements"
---

# Agent P14 Runtime Cognitive Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** 让 `AgentRuntime` 真正持有并调用 `CognitiveCore`，使 P13 上下文文件通过 A00 组装后再进入 C70 MaaS 推理端口。

**Architecture:** 在 `internal/runtime` 定义消费侧 `ContextBuilder` 接口，避免 Runtime 依赖具体 Core 构造细节。`internal/app` 在收到 `ContextPrefix` 时创建 `cognitive.Core` 并注入 Runtime；Runtime 优先使用 CognitiveCore 生成 prompt，缺失时保留旧的直接 prompt 路径。

**Tech Stack:** Go 1.26.0、现有 `internal/cognitive`、`internal/runtime`、`internal/app`、TDD。

---

## 范围

P14 做：

- `AgentRuntime` 增加 `ContextBuilder` 端口。
- Runtime 调 MaaS 前调用 CognitiveCore 组装 prompt。
- App 将 P13 context files block 注入 CognitiveCore，而不是直接拼接 prompt。
- component parity manifest 增加 A00/A01 验收项。

P14 不做：

- 不实现多轮工具调用循环。
- 不改变 C70 MaaS 端口协议。
- 不改变 CLI 的 `--no-context-files` 行为。

## 任务清单

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 验收 |
|---------|--------|------|------|------|------|
| AG-P14-001 | P0 | A01/A00 | Runtime 增加 ContextBuilder 并调用 BuildContext | `done` | MaaS prompt 来自 CognitiveCore |
| AG-P14-002 | P0 | App/P13 | App 将 context files block 注入 CognitiveCore | `done` | CLI context files 仍进入 MaaS prompt |
| AG-P14-003 | P1 | Compat/Docs | parity 与任务表同步 | `done` | A00/A01 纳入 component parity |

## 验证

```powershell
go test ./internal/runtime ./internal/app ./internal/cli ./internal/compat -count=1
go test ./...
go vet ./...
go build -o NUL ./cmd
```
