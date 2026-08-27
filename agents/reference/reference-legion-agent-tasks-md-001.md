---
id: "reference-legion-agent-tasks-md-001"
title: "Legion Agent tasks.md 协作规范"
aliases: ["tasks.md", "任务账本", "多 Agent 任务协作", "Agent handoff"]
type: "reference"
category: "agents/reference"
tags: ["agent", "tasks", "multi-agent", "handoff", "message", "collaboration"]
version: "1.1.0"
created: "2026-05-19"
updated: "2026-08-27"
author: "codex"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "reference-legion-agent-multi-agent-usage-001"
    relation: "related_to"
    path: "./reference-legion-agent-multi-agent-usage-001.md"
  - id: "plans-agent-p21-agent-message-bus-001"
    relation: "precedes"
    path: "../../plans/03-agent/p21-agent-message-bus-plan.md"
---

# Legion Agent tasks.md 协作规范

## 定位

`tasks.md` 是多 Agent 协作的共享任务账本投影视图。它解决的是：

- 多个 Agent 如何发现当前有哪些任务。
- 每个任务由谁负责、谁参与、当前状态是什么。
- 一个 Agent 如何把上下文、问题、结果交接给另一个 Agent。
- 如何避免把长报告、工具输出和聊天记录无限塞进根目录文件。

`tasks.md` 不替代 SQLite session、workflow engine 或 P21 message bus。它是人类可读、Agent 可读、后续可映射到 `agent_messages` 的协作协议。

并发写入时，`tasks.md` 不是源数据。多个 Agent 不应直接读改写 `tasks.md` 后覆盖保存。真正的写入源是 append-only 事件日志：`tasks/events/*.jsonl`。TaskLedger 根据事件日志重建 `tasks.md` 和 `tasks/TASK-*.md`。

## 配置

在 `agent.json` 中使用 `tasks` 配置块约定协作账本路径和膨胀控制规则：

```json
{
  "tasks": {
    "index_path": "tasks.md",
    "root": "tasks",
    "archive_root": "tasks/archive",
    "max_index_lines": 500,
    "max_task_lines": 300,
    "max_message_chars": 300,
    "active_statuses": ["planned", "ready", "in_progress", "blocked", "review"],
    "done_statuses": ["done", "cancelled"]
  }
}
```

字段说明：

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `tasks.index_path` | `tasks.md` | 活跃任务索引投影，只保留摘要和链接 |
| `tasks.root` | `tasks` | 任务账本根目录；`events/*.jsonl` 为事件源，`TASK-*.md` 为详情投影，`.lock` 为重建锁 |
| `tasks.archive_root` | `tasks/archive` | 已完成或取消任务归档投影目录 |
| `tasks.max_index_lines` | `500` | `tasks.md` 最大建议行数，超过后应归档或拆分 |
| `tasks.max_task_lines` | `300` | 单个任务详情文件最大建议行数 |
| `tasks.max_message_chars` | `300` | 写入索引的单条消息摘要最大建议字符数 |
| `tasks.active_statuses` | `planned, ready, in_progress, blocked, review` | 保留在索引中的活跃状态 |
| `tasks.done_statuses` | `done, cancelled` | 应从索引移入归档的终态 |

## 投影、归档与膨胀控制

TaskLedger 重建投影时会执行以下规则：

- `tasks.md` 只显示非终态任务；`done` / `cancelled` 等终态任务不会继续占用活跃看板。
- 终态任务详情会写入 `tasks/archive/TASK-*.md`，并移除 `tasks/TASK-*.md` 中的活跃详情投影。
- `tasks.max_index_lines` 超阈值时，`tasks.md` 会追加截断提示，建议归档终态任务或拆分活跃任务摘要。
- `tasks.max_task_lines` 超阈值时，任务详情会追加截断提示，建议把长内容放到 `docs/` 或 `memory/`，TaskLedger 只保留摘要与链接。
- owner claim 冲突会在任务详情的 `Conflicts` 区显示 `conflict.owner_claim`，并标出 `actor=`、`owner=` 等关键字段。

`/task <task_id>` 读取的是 TaskLedger 事件投影，即使任务已归档，仍可通过任务 ID 查看详情。

可用环境变量覆盖路径和长度限制：

| 环境变量 | 配置字段 |
|----------|----------|
| `LEGION_AGENT_TASKS_INDEX_PATH` | `tasks.index_path` |
| `LEGION_AGENT_TASKS_ROOT` | `tasks.root` |
| `LEGION_AGENT_TASKS_ARCHIVE_ROOT` | `tasks.archive_root` |
| `LEGION_AGENT_TASKS_MAX_INDEX_LINES` | `tasks.max_index_lines` |
| `LEGION_AGENT_TASKS_MAX_TASK_LINES` | `tasks.max_task_lines` |
| `LEGION_AGENT_TASKS_MAX_MESSAGE_CHARS` | `tasks.max_message_chars` |

## 文件分层

推荐结构：

```text
tasks.md
tasks/
  TASK-20260519-001.md
  TASK-20260519-002.md
  events/
    2026-05-19.jsonl
  archive/
    2026-05.md
  .lock
docs/
memory/
logs/
```

分工：

| 文件或目录 | 内容 |
|------------|------|
| `tasks/events/*.jsonl` | append-only 事件源，多个 Agent 只追加不改历史 |
| `tasks.md` | 自动生成的活跃任务看板、owner、参与 Agent、状态、最新摘要、详情链接 |
| `tasks/TASK-*.md` | 自动生成或受控更新的单任务详情、消息线程、交接记录、验收记录 |
| `tasks/archive/` | 完成或取消任务的归档摘要 |
| `tasks/.lock` | TaskLedger rebuild、archive、compact 时使用的锁文件 |
| `docs/` | 长报告、设计文档、说明书、runbook |
| `memory/` | 可复用经验、偏好和长期记忆材料 |
| `logs/` | 工具输出、运行日志、审计细节 |

## 并发写入模型

多 Agent 并行协作时采用事件溯源：

```text
Agent A -> append tasks/events/2026-05-19.jsonl
Agent B -> append tasks/events/2026-05-19.jsonl
TaskLedger -> acquire tasks/.lock
TaskLedger -> replay events
TaskLedger -> rebuild tasks.md and tasks/TASK-*.md
TaskLedger -> release tasks/.lock
```

并发安全验收规则：

- 多个 Agent 可以同时调用 TaskLedger 追加事件；写入会通过 `tasks/.lock` 串行化。
- 事件日志是唯一事实源；重放同一批事件应得到稳定的 `tasks.md` 和 `tasks/TASK-*.md`。
- 详情投影保留完整消息线程，`tasks.md` 只显示当前任务状态、owner、参与者和最新摘要。
- 兼容性测试 `TestTaskLedgerConcurrentCollaborationGolden` 会模拟 researcher、writer、reviewer 并发写入，并用 golden 文件锁定上述行为。

事件示例：

```json
{"event_id":"evt-20260519-000001","task_id":"TASK-20260519-001","type":"message.appended","from":"researcher","to":"writer","summary":"调研完成，证据已写入 docs/research/session-notes.md","artifact":"docs/research/session-notes.md","created_at":"2026-05-19T10:00:00+08:00"}
```

冲突处理：

| 冲突 | 规则 |
|------|------|
| 两个 Agent 同时追加 message | 都保留，按 `created_at` + `event_id` 排序 |
| 两个 Agent 同时 claim owner | 第一个 claim 生效，第二个写入 `conflict.owner_claim` event |
| 两个 Agent 同时更新 status | 后写 wins，但保留 status history |
| `done` 后追加 `blocked` | 不覆盖 `done`，写入 `conflict.late_event`，进入 review |
| 两个 Agent 同时写 result | 都保留为 candidate result，任务进入 `review` |

因此，Agent 工具应该调用 TaskLedger API：

```text
AppendEvent(event)
RebuildIndex()
ReadTask(task_id)
ClaimTask(task_id, agent_id)
UpdateStatus(task_id, status)
ArchiveDone()
```

而不是直接覆盖写 `tasks.md`。

## tasks.md 模板

```markdown
# Agent Tasks

## Rules

- 只保留 active / blocked / review 任务。
- 本文件由 TaskLedger 根据 `tasks/events/*.jsonl` 生成，不手工并发改写。
- 每个任务最多 5-10 行摘要。
- 长内容写入 `tasks/TASK-*.md`、`docs/` 或 `memory/`，这里只放链接。
- `done` 和 `cancelled` 任务应归档到 `tasks/archive/`。

## Agent Roster

| Agent | Role | Profile | Workspace | Notes |
|-------|------|---------|-----------|-------|
| researcher | 调研与证据收集 | review | docs/research | 输出证据和风险 |
| writer | 说明整理与成稿 | fast | docs/writing | 输出用户可读文档 |

## Active Tasks

| Task ID | Status | Owner | Participants | Summary | Detail |
|---------|--------|-------|--------------|---------|--------|
| TASK-20260519-001 | in_progress | researcher | writer | 调研 TUI session 实现并交接给 writer | [[tasks/TASK-20260519-001]] |

## Blocked

| Task ID | Blocker | Waiting For | Since |
|---------|---------|-------------|-------|

## Review Queue

| Task ID | Reviewer | Scope | Detail |
|---------|----------|-------|--------|
```

## 单任务详情模板

```markdown
---
task_id: "TASK-20260519-001"
status: "in_progress"
owner: "researcher"
participants: ["writer"]
created: "2026-05-19"
updated: "2026-05-19"
---

# TASK-20260519-001 调研 TUI session 实现

## Goal

调研当前 TUI session 上下文连续性实现，并把可写入用户手册的结论交接给 writer。

## Context

- Source: `docs/plans/03-agent/p22-session-context-continuity-plan.md`
- Source: `docs/agents/reference/reference-legion-agent-session-001.md`

## Messages

| Time | From | To | Type | Summary | Link |
|------|------|----|------|---------|------|
| 2026-05-19 10:00 | user | researcher | request | 调研当前实现 |  |
| 2026-05-19 10:08 | researcher | writer | handoff | 已确认 session/turn 表和 TUI 命令，可整理说明 | `docs/research/session-notes.md` |

## Deliverables

- `docs/research/session-notes.md`
- `docs/agents/reference/reference-legion-agent-session-001.md`

## Acceptance

- writer 能基于 handoff 独立完成说明。
- 任务详情中包含证据链接。
- `tasks.md` 只保留一行摘要。
```

## 状态协议

| 状态 | 含义 | 谁能设置 |
|------|------|----------|
| `planned` | 已记录但尚未准备执行 | user / coordinator |
| `ready` | 输入足够，等待 Agent 领取 | user / coordinator |
| `in_progress` | Agent 正在处理 | owner |
| `blocked` | 缺少输入、权限、文件或决策 | owner |
| `review` | 已交付，等待审查 | owner / reviewer |
| `done` | 验收通过 | user / reviewer / coordinator |
| `cancelled` | 不再执行 | user / coordinator |

状态流：

```text
planned -> ready -> in_progress -> blocked -> in_progress -> review -> done
planned -> cancelled
ready -> cancelled
```

## 消息类型

| 类型 | 用途 |
|------|------|
| `request` | 请求另一个 Agent 执行任务 |
| `handoff` | 交接上下文、证据、半成品或下一步建议 |
| `result` | 交付结果 |
| `question` | 提出阻塞问题 |
| `answer` | 回复问题 |
| `review` | 审查意见 |
| `decision` | 记录决策 |

每条消息必须包含：

- `from`
- `to`
- `type`
- `summary`
- 必要时附 `link`

不要把完整长文贴到消息表中。超过 `tasks.max_message_chars` 的内容必须写到 `docs/`、`memory/` 或任务详情附录，并在消息里保留摘要和链接。

## Agent 交接规则

一个 Agent 交接给另一个 Agent 时，必须写清：

```markdown
### Handoff: researcher -> writer

- Task: TASK-20260519-001
- Status: review
- Summary: 已完成实现调研，建议 writer 按 session 数据流、TUI 命令、HTTP 查询三段整理。
- Evidence:
  - `internal/storage/sqlite.go`
  - `internal/tui/interactive.go`
  - `docs/agents/reference/reference-legion-agent-session-001.md`
- Next:
  - writer 整理用户手册段落
  - reviewer 检查是否遗漏 HTTP session 查询接口
```

交接不是“聊天”。交接内容必须让接收方不需要翻完整历史也能继续工作。

## 膨胀控制

`tasks.md` 必须保持轻量，并且视为 generated projection：

- 最大建议 500 行。
- 不作为并发写入源。
- 每个活跃任务最多 5-10 行摘要。
- `done`、`cancelled` 任务从索引移入 `tasks/archive/`。
- 长工具输出写入 `logs/` 或任务详情附录。
- 长调研报告写入 `docs/`。
- 长期经验写入 `memory/`。

单任务详情也不能无限追加：

- 最大建议 300 行。
- 消息线程只保留关键消息。
- 连续多轮讨论应合并成一次 `summary`。
- 超长附件用链接引用。

## 与 P21 Message Bus 的关系

`tasks.md` 是 P21 `agent_messages` 的文件态前置规范。

映射关系：

| tasks.md 字段 | P21 候选字段 |
|---------------|--------------|
| `Task ID` | `task_id` |
| `event_id` | `source_event_id` |
| `idempotency_key` | `id`（优先使用；为空时回退到 `event_id`） |
| `Task ID` | `thread_id`（默认同 `task_id`） |
| `From` | `from_agent_id` |
| `To` | `to_agent_id` |
| `Type` | `type`：`message` / `result` / `handoff` / `review` |
| `Summary` | `summary` |
| `Link` / `artifact` | `artifact` |
| `Status` | `status`：`unread` / `read` |
| `created_at` | `created_at` |

当前 P21B 已落地 `AgentMessage` 数据模型、SQLite `agent_messages` 表、`send_message` / `read_messages` 工具、TUI `/send` / `/inbox` 人工入口，以及 HTTP `GET/POST /v1/agents/{id}/messages` 管理面。模型和外部系统都可创建消息、按 recipient/status/task 查询、用 `mark_read=true` 标记已读，并通过 `source_event_id` 追溯 P21A TaskLedger event。

工具参数概要：

| 工具 | 关键参数 | 说明 |
|------|----------|------|
| `send_message` | `to`, `summary`, `from`, `task_id`, `type`, `artifact` | 向目标 Agent inbox 写入一条 `AgentMessage` |
| `read_messages` | `to`, `from`, `status`, `task_id`, `limit`, `mark_read` | 读取 inbox/outbox；`mark_read=true` 会把返回的 unread 消息标记为 read |

TUI `/send <agent> <message>` 会向目标 Agent 写入 unread `AgentMessage`；`/inbox` 默认查看当前 Agent 的未读消息；`@agent --inbox ...` 会把目标 Agent 的 unread 消息注入 prompt，并在目标 Agent 成功运行后标记为 read。文件规范仍可作为人类可读的协作视图和导出格式，`tasks/events/*.jsonl` 可作为 SQLite message bus 的导入源或审计导出格式。

## 相关文档

- [[reference-legion-agent-tools-001|工具能力]] — TaskLedger 与 AgentMessage 工具
- [[reference-legion-agent-multi-agent-usage-001|多 Agent 调用]] — `--task` 绑定与消息交接
- [[reference-legion-agent-config-context-001|配置与上下文文件]] — `tasks.*` 配置
- [[multi-agent-collaboration|多 Agent 协作]] — workflow 与协作方式对比
