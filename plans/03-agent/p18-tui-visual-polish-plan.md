---
id: "plans-agent-p18-tui-visual-polish"
title: "Agent P18 TUI 视觉与交互增强计划"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "tui", "bubble-tea"]
version: "0.1.0"
created: "2026-05-17"
updated: "2026-05-17"
status: "done"
---

# Agent P18 TUI Visual Polish Plan

## 目标

P17 已经完成真正交互式 `agent tui`，但界面仍接近普通文本输出。P18 将 TUI 入口增强为更明确的 Bubble Tea 体验：

- 进入 `agent tui` 时使用 alternate screen，终端切换到全屏缓冲区，达到清屏进入和退出恢复的效果。
- 使用 Lip Gloss 样式组织标题、输入、结果、事件流、审计和状态栏。
- 保留 P13 上下文文件加载能力，继续加载 `AGENTS.md`、`MEMORY.md`、`SOUL.md`、`TOOLS.md`、`USER.md`。

## 任务清单

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 验收 |
|---------|--------|------|------|------|------|
| AG-P18-001 | P0 | CLI/TUI | `agent tui` 启用 Bubble Tea alternate screen | `done` | 进入 TUI 时清屏，退出后恢复终端 |
| AG-P18-002 | P0 | TUI | 交互视图改为分区面板和状态栏 | `done` | 显示 PROMPT、RESULT、EVENT STREAM、AUDIT、STATUS |
| AG-P18-003 | P1 | Tests/Docs | 补充测试并同步计划文档 | `done` | TUI 专项测试覆盖布局关键区域 |

## 验证命令

```powershell
go test ./internal/tui -run TestInteractiveModel -count=1
go test ./internal/tui ./internal/cli -count=1
go test ./...
go vet ./...
go build -o NUL ./cmd
```

## 使用方式

```powershell
go run ./cmd -- tui --config .\agent.json
```

进入后直接输入 prompt，按 Enter 执行；按 Esc 或 Ctrl+C 退出。
