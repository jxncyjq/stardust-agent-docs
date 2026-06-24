---
id: "plans-agent-p22-session-context-continuity-001"
title: "P22 Session Context Continuity 会话上下文连续性计划"
type: "implementation-plan"
category: "plans/agent"
tags: ["agent", "session", "conversation", "context", "tui", "memory"]
version: "0.1.0"
created: "2026-05-19"
updated: "2026-05-19"
status: "done"
related_docs:
  - path: "./p21-agent-message-bus-plan.md"
    relation: "complements"
  - path: "./p13-runtime-context-memory-plan.md"
    relation: "builds_on"
---

# P22 Session Context Continuity 会话上下文连续性计划

## 背景

当前 TUI 只有进程内 `turns` 展示与 `/history`，每次 Enter 都是一个独立 task。Runtime 会重新加载 `AGENTS/SOUL/TOOLS/USER/MEMORY`，但不会自动把最近几轮用户问题和模型回答注入下一次 prompt。

这意味着当前系统有静态上下文和持久化 task/event/audit，但还没有完整的 session/thread 机制。

## 目标

P22 目标是补齐用户与 Agent 的多轮会话连续性：

- 定义 `AgentSession` 与 `ConversationTurn` 数据模型。
- TUI 每次输入绑定 `session_id`。
- 保存 user/agent turn 到 SQLite。
- 下一轮调用自动注入最近 N 轮对话。
- 支持 TUI session 命令：`/new`、`/sessions`、`/switch`、`/clear-session`。
- 支持多 Agent 在同一 session/thread 中协作，保留 `agent_id` 分支信息。

## 非目标

- 不替代 P21 Agent Message Bus。
- 不实现长期语义记忆的自动蒸馏，P22 只做 session 短期上下文连续。
- 不要求跨设备同步。
- 不把完整无限历史直接塞入 prompt，必须有窗口和截断策略。

## P21 与 P22 边界

| 能力 | 归属 | 说明 |
|------|------|------|
| Agent A 给 Agent B 发消息 | P21 | inbox/outbox、message bus、send/read message 工具 |
| 用户与同一 Agent 多轮对话连续 | P22 | session、turn、最近 N 轮上下文注入 |
| `@researcher` 与 `@writer` 共享同一话题历史 | P22 | 同一 session 内保留不同 agent_id 的 turn |
| Workflow 前序结果传递给后序 task | P21 | result handoff 属于编排消息传递 |
| 长期记忆沉淀到 MEMORY.md/Gene/Capsule | 后续 P23+ | 需要质量门控和压缩策略 |

## 任务拆分

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 验收 |
|---------|--------|------|------|------|------|
| AG-P22-001 | P0 | Domain/Storage | 定义 AgentSession 与 ConversationTurn 持久化 | `done` | 可创建 session、追加 turn、按 session 查询 turns |
| AG-P22-002 | P0 | TUI | TUI 启动创建/恢复当前 session | `done` | TUI footer/header 显示 session，重启可恢复最近 session |
| AG-P22-003 | P0 | Runtime/A00 | 最近 N 轮 turn 注入 CognitiveCore prompt | `done` | 第二轮请求包含上一轮用户问题与 Agent 回复摘要 |
| AG-P22-004 | P1 | TUI Command | `/new`、`/sessions`、`/switch`、`/clear-session` | `done` | 用户可创建、切换、清空会话 |
| AG-P22-005 | P1 | Multi-Agent | session turn 记录 agent_id 与 model profile | `done` | `@researcher`、`@writer` 在同一 session 中可追踪各自输出 |
| AG-P22-006 | P1 | Context Policy | 上下文窗口、截断和敏感信息过滤 | `done` | 最近 N 轮可配置，超长内容被安全截断 |
| AG-P22-007 | P1 | API/Docs | HTTP session 查询接口与文档同步 | `done` | 可查询 session/turn，配置文档说明 session 行为 |

## 最小用户体验

### 默认连续对话

```text
你是什么模型
那你刚才说的第三点展开讲讲
```

第二轮应能看到第一轮的问题和回答摘要。

### 多 Agent 同一 session

```text
@researcher 调研当前实现
@writer 根据 researcher 的调研结果整理说明
```

P22 目标是让 writer 能获得同一 session 中 researcher 的上一轮输出，而不是只依赖用户复制。

### 会话命令

```text
/new
/sessions
/switch session-20260519-001
/clear-session
```

## 配置草案

```json
{
  "session": {
    "enabled": true,
    "default_recent_turns": 6,
    "max_turn_chars": 6000,
    "restore_latest_on_tui_start": true
  }
}
```

## 验证计划

- `go test ./internal/storage ./internal/cognitive ./internal/tui ./internal/cli -count=1`
- `go test ./...`
- 手动 smoke：
  - 启动 TUI，连续两轮问答，第二轮要求引用第一轮内容。
  - 使用 `/new` 创建新 session，确认历史不再注入。
  - 使用 `@researcher` 后再 `@writer`，确认同一 session 中 turn 可被后续 Agent 使用。
