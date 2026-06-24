---
id: "plans-agent-sprint-01-scaffold-001"
title: "Agent Sprint 01 Scaffold 计划"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "sprint", "scaffold"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# Agent Sprint 01 Scaffold 计划

## 目标

创建 `agent` 独立 Go 仓库骨架，让 `go test ./...` 通过，并让 `go run ./cmd` 能执行一个 fake task。终端交互入口使用 Bubble Tea TUI 展示 Agent 运行介绍和 fake task 执行过程。

## 任务清单

| 顺序 | 任务 | 验收 |
|------|------|------|
| 1 | 创建 `agent/go.mod`，Go 1.26.0 | `go env GOVERSION` 与 go.mod 兼容 |
| 2 | 创建 `agent/cmd/main.go` | 使用 `run() error` 模式 |
| 3 | 建立 `internal/app` 装配 | 能创建 fake runtime |
| 4 | 建立 `internal/domain` 类型 | Agent、Task、ToolCall、ApprovalTicket 编译通过 |
| 5 | 建立 `internal/port` 端口 | Maas/EventBus/AuditLog/Knowledge 接口有 fake |
| 6 | 建立 `internal/task` | `TryLock` 并发测试通过 |
| 7 | 建立 `internal/runtime` | fake MaaS 返回结果，任务完成 |
| 8 | 建立 `internal/approval` | HardLoop 工单创建/批准/拒绝测试通过 |
| 9 | 建立 `internal/quality` | fake Aegis 审核通过 |
| 10 | 建立 `internal/cli` 与 `internal/tui` | `agent run --demo` 使用 Bubble Tea 展示 fake task 状态 |
| 11 | 建立 Makefile | `make test`、`make build` 可用 |

## 完成定义

- `go test ./...` 通过。
- `go run ./cmd -- run --demo` 能通过 Bubble Tea TUI 展示一个 fake task 的执行结果。
- `go run ./cmd -- run --demo --plain` 能输出非交互日志，便于脚本和 CI 验证。
- 没有真实外部服务依赖。
- 没有 `util/common/helper` 泛包。
