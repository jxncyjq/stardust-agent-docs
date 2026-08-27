---
id: "reference-legion-agent-session-001"
title: "Legion Agent 会话连续性"
aliases: ["session", "会话连续", "conversation turns"]
type: "reference"
category: "agents/reference"
tags: ["agent", "session", "sqlite", "context", "conversation"]
version: "2.0.0"
created: "2026-05-19"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "agent-sqlite-schema-001"
    relation: "related_to"
    path: "../legion-agent/sqlite-schema.md"
  - id: "reference-legion-agent-backend-api-001"
    relation: "related_to"
    path: "./reference-legion-agent-backend-api-001.md"
---

# Legion Agent 会话连续性

## 工作方式

开启 `session.enabled` 且使用 SQLite 时，TUI 会把每轮用户输入和模型回复写入 `agent_sessions` 与 `conversation_turns`。下一轮任务会读取最近 N 轮 turn 并注入 prompt，从而保持上下文连续。

数据流如下：

```text
用户在 TUI 输入
  -> runTUITask 读取当前 session 最近 turns
  -> Runtime/CognitiveCore 将 Recent conversation 注入 prompt
  -> 模型返回结果
  -> RecordExchange 写入 user turn 和 assistant turn
  -> session context cache 失效，下一轮重新读取最新窗口
```

每条 turn 会记录：

| 字段 | 作用 |
|------|------|
| `session_id` | 当前会话 |
| `task_id` | 触发本轮对话的任务 |
| `agent_id` | 实际回答的 Agent |
| `model_profile` | 使用的模型 profile |
| `role` | `user` 或 `assistant` |
| `content` | 已按 `max_turn_chars` 截断的内容 |
| `prompt_tokens` / `completion_tokens` / `cached_tokens` / `total_tokens` | 该轮 token 消耗（老数据为 0，只有新会话有值） |

## 推荐配置

```json
{
  "storage": {
    "driver": "sqlite",
    "path": "agent.db"
  },
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

## Session Context Cache

`session.cache_enabled` 开启后，TUI 会把当前 session 最近 N 轮、已按 `max_turn_chars` 截断后的上下文窗口缓存在进程内。它只缓存可注入 Runtime 的短期上下文窗口，用于减少重复读取和重复整理成本。

它不是长期记忆系统：

| 能力 | session cache | memory |
|------|---------------|--------|
| 生命周期 | 当前 Agent 进程内 | 文件或数据库持久保存 |
| 内容 | 最近 N 轮 conversation turns | 经验、偏好、知识、总结 |
| 失效 | 新 turn、切换 session、新建 session、清空 session | 由记忆写入、归并或清理策略决定 |
| 目标 | 加速上下文窗口读取 | 让 Agent 跨任务学习和复用经验 |

如果你正在调试上下文是否实时刷新，可以临时设置：

```json
{
  "session": {
    "cache_enabled": false
  }
}
```

## TUI 会话命令

| 场景 | 操作 |
|------|------|
| 开始新主题 | `/new` |
| 查看历史会话 | `/sessions` |
| 回到旧主题 | `/switch <session_id>` |
| 当前会话上下文污染 | `/clear-session` |

典型用法：

```text
/sessions
/switch session-1770000000000000000
/new
```

`/new` 和 `/clear-session` 都会创建新 session。区别是语义：`/new` 用于开始新主题，`/clear-session` 用于当前上下文污染时快速切断历史。

## 从 session 恢复对话

默认配置 `restore_latest_on_tui_start=true` 时，TUI 启动会自动恢复当前 Agent 最近一次 session：

```powershell
go run ./cmd/agent -- tui --config .\agent.json
```

进入后可以直接继续提问，Agent 会读取最近 `default_recent_turns` 轮 turn 并注入上下文。

如果要恢复指定历史会话：

```text
/sessions
/switch session-1770000000000000000
继续刚才关于 session cache 的讨论，整理剩余风险
```

推荐恢复流程：

1. 用 `/sessions` 找到目标 session。
2. 用 `/switch <session_id>` 切换。
3. 先问一句“请概括当前会话上下文”，确认恢复到了正确主题。
4. 再继续具体任务。

如果只想查询历史，不想进入 TUI，可以使用 HTTP：

```powershell
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions?company_id=cli-company&agent_id=cli-agent"
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions/session-1770000000000000000/turns?limit=12"
```

HTTP 查询接口只读取 session/turn，不会自动把历史恢复到某个正在运行的 TUI。真正继续对话仍建议通过 TUI `/switch`。

## 工作模式与工作目录

会话上除了历史，还挂着两个决定任务行为的字段：

| 字段 | 取值 | 影响 |
|------|------|------|
| `mode` | `auto`（默认）/ `plan` / `manual` | 该会话派生的每个任务都继承它：`plan` 只给只读工具，`manual` 把 Sensitive 工具挡在人工审批后 |
| `working_dir` | 绝对目录 | 决定工具沙箱根与 `agents.md` 项目根，任务创建时继承 |

TUI 用 `/mode`、`/cwd` 设置，HTTP 用 `POST /v1/sessions` 或 `PATCH /v1/sessions/{id}`。

两条硬约束：

- `working_dir` **一次性可设**：空 → 有值可以，重设同值可以，改成另一个目录直接拒绝（TUI 报错 / HTTP 400）。会话磁盘状态按写入时的目录归档，改指向会遗弃既有状态。
- 提交任务时若会话的 `working_dir` 指向不存在的目录，`POST /v1/tasks` 返回 400，而不是让任务落到错误的根目录上跑。

## Session 与 Workflow 的关系

Session 负责 TUI 对话连续性；Workflow 负责服务端 task 编排。二者不是同一个状态：

| 能力 | Session | Workflow |
|------|---------|----------|
| 入口 | `agent tui`、`/sessions`、`/switch` | `agent serve`、`POST /v1/workflows` |
| 目标 | 恢复人机对话上下文 | 编排多个 task/Agent |
| 持久化 | `agent_sessions`、`conversation_turns` | workflow state、task、event、audit |
| 上下文传递 | 最近 N 轮 turn 注入 prompt | `{{tasks.<task_id>.result}}`、TaskLedger、message bus |

当前 workflow 不会自动“挂载某个 TUI session”。如果想让 workflow 基于历史 session 继续工作，有两种做法：

1. 通过 `/v1/sessions/{session_id}/turns` 取出历史 turns，把关键内容摘要写入 workflow `task.input`。
2. 在 TUI 中 `/switch <session_id>` 后，用 `@agent --task` 或 `/send` 让子 Agent 接着处理。

示例：先查询 session turns，再提交 workflow：

```powershell
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions/session-1770000000000000000/turns?limit=8"
```

把得到的关键上下文整理进 workflow task input：

```json
{
  "id": "wf-from-session",
  "root": {
    "id": "write-from-session",
    "kind": "agent_task",
    "task": {
      "id": "write-from-session-1",
      "agent_id": "writer",
      "input": "基于 session-1770000000000000000 最近对话摘要：<粘贴摘要>，整理一份用户说明。"
    }
  }
}
```

## 多 Agent 会话

`@researcher`、`@writer` 等子 Agent 会共享当前 TUI session，但每条 assistant turn 会记录真实 `agent_id` 和 `model_profile`。这意味着 writer 可以看到 researcher 在同一 session 中刚刚输出的内容，同时历史中仍能区分是谁回答的。

示例：

```text
@researcher 调研当前 cache 实现
@writer 基于 researcher 的上一轮输出整理说明
```

如果需要更可靠的跨 Agent 交接，建议使用 `--task` 或 `--inbox`，见 [[reference-legion-agent-multi-agent-usage-001|多 Agent 调用]]。

## 会话的 HTTP 面

`agent serve` 在 SQLite 存储模式下把会话暴露为完整的 CRUD 面，不只是只读查询：

| 端点 | 用途 |
|------|------|
| `POST /v1/sessions` | 新建会话，可带 `mode`、`working_dir`；id 由服务端生成 |
| `GET /v1/sessions` | 会话列表，按最近更新倒序，支持 `company_id`、`agent_id` |
| `GET /v1/sessions/{id}/turns` | 会话 turns，`limit=0` 或省略即全量 |
| `PATCH /v1/sessions/{id}` | 改 `title` / `project` / `archived` / `mode` / `working_dir` |
| `DELETE /v1/sessions/{id}` | 删除会话行，并删掉它在磁盘上的会话目录 |

```powershell
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions?company_id=cli-company&agent_id=cli-agent"
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/sessions/session-123/turns?limit=6"
```

两个契约细节：

- 请求体里出现服务端不认识的字段会被 **400 拒绝**，不会被静默丢弃（典型如客户端自带 `id`）。
- 磁盘目录删除失败时返回 500 而不是「删除成功」——库里行没了、磁盘还留着残留状态，这必须让调用方看见。

HTTP 面只读/改数据，不会把历史「恢复」到某个正在运行的 TUI；继续对话仍走 TUI `/switch`。

## 排查

上下文没有连续时，按顺序检查：

1. `storage.driver` 是否为 `sqlite`。
2. `session.enabled` 是否为 `true`。
3. `session.default_recent_turns` 是否大于 0。
4. 是否刚执行过 `/new` 或 `/clear-session`。
5. 当前是否切换到了另一个 session。
6. `agent.db` 是否被删除或恢复到了旧备份。

## 相关文档

- [[reference-legion-agent-config-context-001|配置与上下文文件]] — `session.*` 与 `workspace.*` 配置
- [[reference-legion-agent-tui-001|TUI 使用]] — `/new`、`/sessions`、`/switch`、`/mode`、`/cwd`
- [[reference-legion-agent-backend-api-001|后端系统调用参考]] — 会话与任务端点
- [[reference-legion-agent-multi-agent-usage-001|多 Agent 调用]] — 同一会话内的多 Agent 协作
- [[agent-sqlite-schema-001|agent.db 数据结构]] — `agent_sessions` / `conversation_turns` 表
