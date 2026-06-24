---
id: "plans-agent-p23-session-context-cache-001"
title: "P23 Session Context Cache 计划"
type: "plan"
category: "plans/agent"
tags: ["agent", "session", "cache", "context", "tui"]
version: "0.1.0"
created: "2026-05-25"
updated: "2026-05-25"
status: "done"
---

# P23 Session Context Cache 计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:test-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 P22 session continuity 之上补齐可命中、可失效、可观测的 session context cache，降低重复读取和重复上下文整理成本。

**Architecture:** P23 不替代长期 memory，也不做语义记忆蒸馏；它缓存的是“当前 session 最近 N 轮、已截断、已净化后可注入 Runtime 的上下文窗口”。第一批采用进程内内存 cache，后续再扩展 SQLite 持久化 cache、摘要压缩和 token budget 驱逐。

**Tech Stack:** Go 1.26、SQLite session/turn 存储、TUI session controller、Runtime/CognitiveCore prompt 注入、标准库 `sync`。

---

## 范围

### 本批完成

- 定义 session context cache 的配置项。
- 提供线程安全的内存 cache。
- TUI session controller 的 `RecentTurns` 使用 cache。
- 在 `/new`、`/switch`、`/clear-session`、`RecordExchange` 后失效当前 session cache。
- 暴露基础 stats，方便后续接入 diagnostics/metrics。
- 更新参考手册与计划索引。

### 暂不完成

- 不做跨进程共享 cache。
- 不做 SQLite 持久化 context cache。
- 不做自动摘要压缩和长期 memory 蒸馏。
- 不做向量召回或语义 cache。

## 任务表

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P23-001 | P0 | Config/Plan | 增加 session cache 配置与计划索引 | `done` | `internal/config`, `docs/plans` | `session.cache_enabled`、`session.cache_max_entries` 可加载 |
| AG-P23-002 | P0 | Cache | 实现内存 SessionContextCache | `done` | `internal/sessioncache` | 支持 get/put/invalidate/stats，返回副本且线程安全 |
| AG-P23-003 | P0 | TUI/CLI | `RecentTurns` 接入 cache 和失效策略 | `done` | `internal/cli` | 连续读取命中 cache，追加 turn 后失效 |
| AG-P23-004 | P1 | Docs | 更新 session/cache 使用说明 | `done` | `docs/agents/reference` | 手册说明 cache 与 memory 的边界 |
| AG-P23-005 | P1 | Verification | 兼容性与总验收 | `done` | `internal/compat`, `docs/plans` | `go test ./...`、`go vet ./...`、`go build -o NUL ./cmd` 通过 |

## 配置草案

```json
{
  "session": {
    "enabled": true,
    "default_recent_turns": 6,
    "max_turn_chars": 6000,
    "restore_latest_on_tui_start": true,
    "cache_enabled": true,
    "cache_max_entries": 128
  }
}
```

## Cache Key

第一批 key 由以下字段组成：

- `company_id`
- `agent_id`
- `session_id`
- `model_profile`
- `recent_turns`
- `max_turn_chars`

后续如果 context files、summary 或 token budget 进入 cache，key 需要增加 `context_files_hash`、`policy_version` 和 `model_context_window`。

## 验收脚本

```powershell
go test ./internal/sessioncache ./internal/cli ./internal/config -count=1
go test ./...
go vet ./...
go build -o NUL ./cmd
```
