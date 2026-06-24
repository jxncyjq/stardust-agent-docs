---
id: "plans-agent-p21-agent-message-bus-001"
title: "P21 Agent Message Bus 与 Agent 通讯计划"
type: "implementation-plan"
category: "plans/agent"
tags: ["agent", "multi-agent", "message-bus", "communication", "tui", "workflow"]
version: "0.1.0"
created: "2026-05-19"
updated: "2026-05-25"
status: "completed"
related_docs:
  - path: "./p20-multi-agent-runtime-routing-plan.md"
    relation: "builds_on"
  - path: "../../agents/reference/multi-agent-collaboration.md"
    relation: "updates"
  - path: "../../agents/reference/reference-legion-agent-tasks-md-001.md"
    relation: "uses"
---

# P21 Agent Message Bus 与 Agent 通讯计划

## 背景

P20 已完成 `task.agent_id` 到不同 Agent runtime 的路由，TUI 也可通过 `@researcher ...`、`@writer ...` 调用不同 Agent。

当前仍缺少真正的 Agent 间通讯能力：一个 Agent 的输出不会自动进入另一个 Agent 的 inbox，也没有标准的 send/reply/handoff 协议。现阶段协作仍主要依赖人工复制、共享文件或 workflow 编排。

P21 实现前，文件态协作协议以 TaskLedger 为准：`tasks/events/*.jsonl` 是唯一写入源，根目录 `tasks.md` 和 `tasks/TASK-*.md` 是可重建投影。该规范用于约束不同 Agent 在消息总线落地前如何交换信息，并作为后续 `agent_messages` 表和 `/send`、`/inbox` 的字段映射参考。

## 目标

P21 目标是补齐最小可用的 Agent 通讯闭环：

- 提供 Agent Message 数据模型与存储。
- Runtime 工具支持 `send_message` 与 `read_messages`。
- TUI 支持 `/send`、`/inbox` 和基于 `@agent` 的消息式协作。
- Workflow 可把前一个 task 结果作为后续 task 的输入或 message。
- 文档明确“任务路由”和“Agent 通讯”的边界。
- 文档明确 `tasks.md` 文件态协作账本与 SQLite message bus 的映射关系。

## 非目标

- 不实现跨机器分布式消息队列。
- 不实现组织层级、预算、雇佣、复杂权限审批。
- 不替代现有 workflow engine；P21 只补齐 Agent 间消息与结果传递。
- 不要求所有消息都由 LLM 自动处理，第一版允许人工在 TUI 中触发读取与继续处理。

## 任务拆分

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 验收 |
|---------|--------|------|------|------|------|
| AG-P21-000 | P0 | Docs/Config | `tasks.md` 文件态协作配置与规范 | `done` | 已新增 `tasks` 配置块、参考手册和 P21 映射说明 |
| AG-P21-F01 | P0 | Domain/File | 实现 TaskLedger append-only event store 与投影服务 | `done` | 已新增事件 schema、append-only JSONL、锁、幂等 replay、deterministic projection、并发追加和路径边界测试 |
| AG-P21-F02 | P0 | Tool/Runtime | 新增 `create_task`、`claim_task`、`update_task`、`append_task_message`、`read_task`、`rebuild_tasks` 工具 | `done` | 已注入 CLI/TUI/serve/子 Agent Runtime，模型通过真实工具追加事件和读取投影，不直接手写 `tasks.md` |
| AG-P21-F03 | P0 | TUI | 新增 `/tasks`、`/task <id>`、`/handoff <agent> <task_id> <summary>` 命令 | `done` | TUI 已可查看任务看板、读取详情、发起 handoff 并显示更新后的任务投影 |
| AG-P21-F04 | P1 | TUI/@Agent | `@agent --task` 任务绑定 | `done` | `@researcher --task TASK-*` / `@writer --task TASK-*` 已注入 TaskLedger 任务详情，模型结果会追加为 `result.appended` 并重建投影 |
| AG-P21-F05 | P1 | Guard/Archive | tasks 并发冲突、膨胀控制与归档执行 | `done` | owner claim 冲突已在详情投影显示 actor/owner；超阈值会提示归档或拆分；done/cancelled 已投影到 `tasks/archive/` 并移除活跃详情投影 |
| AG-P21-F06 | P1 | Docs/Compat | 文件态 tasks 并发协作 smoke 与文档同步 | `done` | compat golden smoke 已覆盖 replay deterministic、并发追加消息均保留、索引不泄漏消息历史，参考文档已同步 |
| AG-P21-001 | P0 | Domain/Storage | 定义 AgentMessage 数据模型与 SQLite 持久化 | `done` | 已新增 AgentMessage/Query/TaskEventFields 映射与 SQLite `agent_messages`，支持创建、按 recipient/status/task 查询、标记已读 |
| AG-P21-002 | P0 | Port/Tool | 新增 `send_message` / `read_messages` 工具 | `done` | Runtime 已可由模型调用工具发送消息、读取 inbox，并可通过 `mark_read` 标记已读 |
| AG-P21-003 | P0 | TUI | 增加 `/send <agent> <message>` 与 `/inbox` 命令 | `done` | TUI 已可查看当前 Agent 未读 inbox，并向目标 Agent 写入持久化 `AgentMessage` |
| AG-P21-004 | P1 | TUI/@Agent | 扩展 `@agent` 调用为可选 message handoff | `done` | `@writer --inbox 根据 researcher 最新消息整理` 可读取目标 Agent 未读消息，成功运行后标记已读 |
| AG-P21-005 | P1 | Workflow | 支持 task result 传递到后续 task input/message | `done` | 后续 task input 已可通过 `{{tasks.<task_id>.result}}` 引用 `task_completed` 事件中的前序 task result |
| AG-P21-006 | P1 | API | HTTP 管理面增加 message 查询/发送接口 | `done` | `GET/POST /v1/agents/{id}/messages` 可发送、查询和 `mark_read`，OpenAPI golden 已同步 |
| AG-P21-007 | P1 | Docs/Compat | 更新多 Agent 协作文档与兼容性测试 | `done` | 文档已区分 routing、session、TaskLedger、message bus、workflow handoff、HTTP message API 六类协作面，并新增 P21 collaboration surface compat golden |
| AG-P21-008 | P1 | Docs/Config | 将 `tasks.md` 协作规范纳入配置和参考手册 | `done` | `tasks` 配置块、`reference-legion-agent-tasks-md-001.md` 和 reference index 已同步 |

## P21A 文件态 tasks 协作前置批次

P21A 先补齐基于 `tasks.md` 的本地文件态协作能力，再进入 SQLite `agent_messages`。目标是让不同 Agent 在没有完整 message bus 的情况下，也能通过标准任务账本交换信息。

### P21A 范围

- 读取 `config.Tasks` 中的 `index_path`、`root`、`archive_root` 和长度限制。
- 提供 TaskLedger 服务，负责追加事件、读取投影、重建索引、更新状态、归档任务。
- 将 `tasks.md` 和 `tasks/TASK-*.md` 定义为 generated projection，不允许 Agent 直接并发改写。
- 将 `tasks/events/YYYY-MM-DD.jsonl` 定义为唯一写入源，所有并发更新都以 append-only event 记录。
- 使用 `tasks/.lock` 保护 rebuild / archive / compact 等需要重写投影文件的操作。
- 提供真实工具，避免模型直接手写不规范 markdown。
- TUI 提供最小命令，方便人工发起和查看协作。
- `@agent` 调用支持显式绑定 task id，把结果写成 handoff/result。

### P21A 非目标

- 不实现 SQLite inbox/outbox。
- 不实现跨进程消息投递。
- 不替代 workflow engine。
- 不让 `tasks.md` 保存完整模型输出、工具输出或长报告。
- 不允许普通 Agent 直接读改写 `tasks.md` 后覆盖保存；必须通过 TaskLedger API 或工具追加事件。

### P21A 事件与并发模型

文件态协作采用事件溯源：

```text
Agent A -> append tasks/events/2026-05-19.jsonl
Agent B -> append tasks/events/2026-05-19.jsonl
TaskLedger -> acquire tasks/.lock
TaskLedger -> replay events
TaskLedger -> rebuild tasks.md and tasks/TASK-*.md
TaskLedger -> release tasks/.lock
```

文件角色：

| 文件 | 角色 | 写入规则 |
|------|------|----------|
| `tasks/events/*.jsonl` | append-only source of truth | Agent 只追加，不改历史 |
| `tasks.md` | generated projection | TaskLedger 重建，Agent 不直接手改 |
| `tasks/TASK-*.md` | generated task detail projection | TaskLedger 重建或受控追加 |
| `tasks/.lock` | rebuild/archive lock | 只有 TaskLedger 获取 |

事件最小字段：

```json
{
  "event_id": "evt-20260519-000001",
  "task_id": "TASK-20260519-001",
  "type": "message.appended",
  "schema_version": 1,
  "from": "researcher",
  "to": "writer",
  "actor_agent_id": "researcher",
  "correlation_id": "session-20260519-001",
  "idempotency_key": "researcher:TASK-20260519-001:message:0001",
  "summary": "调研完成，证据已写入 docs/research/session-notes.md",
  "artifact": "docs/research/session-notes.md",
  "created_at": "2026-05-19T10:00:00+08:00"
}
```

字段规则：

| 字段 | 规则 |
|------|------|
| `event_id` | 全局唯一，推荐 `evt-YYYYMMDD-HHMMSS-rand`；replay 时按该字段去重 |
| `task_id` | 必须匹配 `TASK-YYYYMMDD-NNN` 或已有任务 ID；不存在时只有 `task.created` 可创建 |
| `type` | 第一批支持 `task.created`、`task.claimed`、`task.status_changed`、`message.appended`、`result.appended`、`handoff.appended`、`review.appended`、`conflict.*` |
| `schema_version` | 第一版固定为 `1`，后续升级通过兼容 reader 处理 |
| `actor_agent_id` | 必须来自根配置 `agents` 或当前根 Agent ID；未知 Agent 追加被拒绝 |
| `correlation_id` | 关联 session、workflow、HTTP request 或 TUI run，便于追踪 |
| `idempotency_key` | 可选但推荐；相同 key 的重复事件 replay 时只保留第一条，并记录 duplicate 诊断 |
| `summary` | 受 `tasks.max_message_chars` 限制；长内容写入 `artifact` 指向的 `docs/` 或 `memory/` 文件 |
| `artifact` | 必须位于 workspace 内，不能越过 `workspace.docs_root`、`workspace.memory_root` 或允许的任务目录 |

原子写入规则：

- append 事件时使用单行 JSONL，每个事件一行，禁止多行 payload。
- 写事件只允许追加，不允许修改历史 event 文件。
- 同进程并发写入由 TaskLedger 内部 mutex 串行化。
- 跨进程写入优先通过 `tasks/.lock` 的 create-exclusive 或平台文件锁保护；无法获取锁时返回可重试错误，不做静默覆盖。
- 重建 `tasks.md` 和 `tasks/TASK-*.md` 时先写临时文件，再原子 rename 替换投影。
- replay 必须可重复执行；同一批 event 多次 replay 生成完全相同的投影内容。

冲突规则：

| 冲突 | 处理 |
|------|------|
| 两个 Agent 同时追加 message | 都保留，按 `created_at` + `event_id` 排序 |
| 两个 Agent 同时 claim owner | 第一个 claim 生效，第二个写入 `conflict.owner_claim` event |
| 两个 Agent 同时更新 status | 后写 wins，但保留 status history |
| `done` 后追加 `blocked` | 不覆盖 `done`，写入 `conflict.late_event`，进入 review |
| 两个 Agent 同时写 result | 都保留为 candidate result，任务进入 `review` |

### P21A 最小用户体验

```text
/tasks
/task TASK-20260519-001
/handoff writer TASK-20260519-001 已完成调研，请整理成说明
@researcher --task TASK-20260519-001 调研当前实现
@writer --task TASK-20260519-001 根据 handoff 整理说明
```

### P21A 验收

- `tasks.md` 不存在时可初始化标准模板。
- 创建任务会先追加 `task.created` event，再由 rebuild 生成 `tasks/TASK-*.md` 并在 `tasks.md` 增加一行摘要。
- `append_task_message` 只追加 `message.appended` event；rebuild 后详情文件包含消息，索引只保留最新摘要。
- `@researcher --task ...` 执行完成后可追加 `result` 或 `handoff`。
- `@writer --task ...` 能读取任务详情，以 `TaskLedger Context` 块进入 prompt，并受 `max_task_lines`、`max_message_chars` 和安全净化限制。
- 两个 Agent 并发追加消息不会互相覆盖。
- owner claim、late event、candidate result 冲突都有可追踪 conflict event。
- 超过 `max_index_lines` 或 `max_task_lines` 时不会继续静默膨胀，而是提示归档或拆分。
- 重复 `event_id` 或 `idempotency_key` 不会重复投影。
- 修改 event 顺序后 replay 仍按 `created_at + event_id` 生成稳定结果。
- tasks 工具拒绝 workspace 越界路径、未知 agent、过长摘要和直接覆盖投影的请求。

### P21A 任务执行顺序

1. `AG-P21-F01`：先实现 `internal/taskledger` 领域类型、事件 reader/writer、锁、replay、projection golden 测试。
2. `AG-P21-F02`：再把 TaskLedger 暴露为 A20 工具，并接入 Runtime 工具注册。
3. `AG-P21-F03`：补 TUI `/tasks`、`/task`、`/handoff`，只读取投影或调用 TaskLedger。
4. `AG-P21-F04`：扩展 `@agent --task`，把任务详情注入目标 Agent prompt，把运行结果追加为 event。
5. `AG-P21-F05`：补冲突事件、归档、膨胀控制和人类可读错误。
6. `AG-P21-F06`：补并发 smoke、compat/golden、文档和配置注释同步。

## 最小用户体验

### TUI 发送消息

```text
/send writer 请根据 researcher 的调研结果整理成说明
```

### TUI 查看 inbox

```text
/inbox
```

### Agent 调用工具发送消息

模型可通过工具调用表达：

```json
{
  "name": "send_message",
  "arguments": {
    "to": "writer",
    "subject": "调研完成",
    "content": "我已完成当前实现调研，建议从 TUI routing 和 message bus 两层说明。"
  }
}
```

## 设计边界

P20 的 `task.agent_id` 解决“这项任务由谁执行”。

P21 的 `agent_messages` 解决“Agent 之间如何传递上下文、请求、结果和问题”。

`tasks.md` 解决“消息总线尚未实现或需要人类可读视图时，Agent 如何通过文件交换任务状态、摘要和交接材料”。

二者关系：

```text
Workflow/TUI/HTTP
  -> task.agent_id 路由执行者
  -> Agent runtime
  -> send_message/read_messages
  -> agent_messages
  -> 另一个 Agent 后续读取并继续处理
```

## 验证计划

- P21A:
  - `go test ./internal/taskledger ./internal/tool ./internal/tui ./internal/cli ./internal/compat -count=1`
  - compat golden smoke：
    - `TestTaskLedgerConcurrentCollaborationGolden`
    - 并发追加 researcher / writer / reviewer 事件
    - 验证 replay deterministic、消息不丢失、`tasks.md` 只保留任务摘要
  - 手动 smoke：
    - `/tasks` 初始化并显示空看板
    - `/handoff writer TASK-20260519-001 ...`
    - `@writer --task TASK-20260519-001 根据 handoff 继续`
    - 验证 `tasks.md` 只保留摘要，详情写入 `tasks/TASK-*.md`
    - 并发追加两条 event 后 rebuild，验证两条消息都保留且没有整文件覆盖
- `go test ./internal/storage ./internal/tool ./internal/runtime ./internal/tui ./internal/cli ./internal/server -count=1`
- `go test ./...`
- 手动 smoke：
  - TUI 中 `/send writer hello`
  - 切换或调用 `@writer --inbox 读取 inbox 并总结`
  - 验证 message 被写入、读取，模型成功后标记已读，模型失败时保持未读可重试
