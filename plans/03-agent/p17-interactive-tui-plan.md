---
id: "plans-agent-p17-interactive-tui-001"
title: "Agent P17 交互式 TUI 计划"
type: "implementation-plan"
category: "plans/agent"
tags: ["plan", "agent", "p17", "tui", "interactive", "context-files"]
version: "0.1.0"
created: "2026-05-17"
updated: "2026-05-17"
status: "done"
related_docs:
  - path: "./task-breakdown.md"
    relation: "updates"
---

# Agent P17 Interactive TUI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** 新增真正交互式 `agent tui` 命令，启动后可输入 prompt、回车运行任务、展示结果和事件流，并继续停留在界面中。

**Architecture:** 保留现有 `run` 的一次性结果展示模型；新增 `internal/tui.InteractiveModel` 负责输入、执行和结果展示。CLI 新增 `tui` 子命令，复用现有配置、MaaS、SQLite 持久化和 `contextfiles.Load`，确保 `AGENTS.md/MEMORY.md/SOUL.md/TOOLS.md/USER.md` 进入 `ContextPrefix`。

**Tech Stack:** Go 1.26.0、Bubble Tea、现有 `app.RunTask`、P13 context files。

---

## 任务清单

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 验收 |
|---------|--------|------|------|------|------|
| AG-P17-001 | P0 | TUI | 新增 InteractiveModel | `done` | 输入 prompt 后运行任务，界面不自动退出 |
| AG-P17-002 | P0 | CLI/P13 | 新增 `agent tui` 命令并加载五类上下文文件 | `done` | context prefix 包含 AGENTS/MEMORY/SOUL/TOOLS/USER 内容 |
| AG-P17-003 | P1 | Docs/Validation | 计划、任务表和验证同步 | `done` | 测试、vet、build 通过 |

## 验证

```powershell
go test ./internal/tui ./internal/cli -run "TestInteractiveModel|TestBuildRunContextPrefix|TestRootIncludesTUICommand" -count=1
go test ./...
go vet ./...
go build -o NUL ./cmd
```
