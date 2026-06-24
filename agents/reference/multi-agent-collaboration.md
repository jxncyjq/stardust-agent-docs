---
id: "reference-multi-agent-collaboration-001"
title: "多 Agent 协作参考手册"
aliases: ["多 agent", "multi-agent", "并行 agent", "agent 协作", "workflow 并发"]
type: "reference"
category: "agents/reference"
tags: ["multi-agent", "workflow", "parallel", "collaboration", "serve", "tui"]
version: "1.1.0"
created: "2026-05-18"
updated: "2026-05-25"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "reference-configuration-001"
    relation: "related_to"
    path: "../legion-agent/configuration.md"
  - id: "reference-maas-model-profiles-001"
    relation: "related_to"
    path: "../legion-agent/model-profiles.md"
---

# 多 Agent 协作参考手册

<!-- @section: overview -->
## 概述

Legion Agent 提供三种让多个 agent 协作运行任务的方式：

| 方式 | 适用场景 | 实现状态 |
|------|----------|----------|
| **Workflow Engine + per-agent runtime routing** | 在单进程内编排多个 task，并按 `task.agent_id` 切换 Agent 配置 | ✅ 已实现，推荐 |
| **多进程 + 共享文件**（多终端 `agent tui`）| 手动并行，共享 `agent.db` 和文件系统 | ✅ 可用（手动协调）|
| **跨进程路由**（`TaskSpec.AgentID` 分发）| 真正的多 agent 进程互相路由 task | 🚧 单进程内已按 `task.agent_id` 路由；跨进程分发待实现 |
<!-- @end-section -->

---

<!-- @section: collaboration-surfaces -->
## 协作边界

当前本地/单进程多 Agent 协作由六个能力面组成：

| 能力面 | 解决的问题 | 当前状态 |
|--------|------------|----------|
| Runtime routing | 任务由哪个 Agent runtime 执行 | 已完成，`task.agent_id` 路由到不同 Agent 配置、模型 profile、上下文和 workspace |
| Session continuity | 同一话题跨 Agent 对话是否连续 | 已完成，session turn 记录 `agent_id` 与 `model_profile` |
| TaskLedger file collaboration | Agent 如何通过人类可读任务账本协作 | 已完成，`tasks/events/*.jsonl` 为 append-only source，`tasks.md` 和详情文件为投影 |
| Agent message bus | Agent 如何发 inbox/outbox 消息 | 已完成，SQLite `agent_messages`、工具、TUI 和 HTTP API 共用同一数据结构 |
| Workflow result handoff | workflow 前序 task 结果如何传给后续 task | 已完成，支持 `{{tasks.<task_id>.result}}` 占位符 |
| HTTP message API | 外部系统如何发送/查询 AgentMessage | 已完成，`GET/POST /v1/agents/{id}/messages` |

这六个能力面职责不同：routing 只决定执行者，session 负责对话连续性，TaskLedger 负责可读任务账本，message bus 负责 Agent 间消息，workflow handoff 负责编排内结果传递，HTTP API 负责外部集成。
<!-- @end-section -->

---

<!-- @section: workflow-engine -->
## 方式一：Workflow Engine（推荐）

### 原理

`agent serve` 启动后暴露 HTTP 管理 API。设计目标是向 `POST /v1/workflows` 提交 **Workflow Definition JSON** 后，由 Workflow Engine 按节点类型调度 task，支持串行、并发、条件分支、循环、审批挂起等。

当前代码中 `agent serve` 会注入 `workflow.Engine`、`WorkflowEvents` 与后台 `Coordinator`，形成端到端可运行的 workflow 执行闭环。多 Agent 运行时路由的精确边界见 [[spec-multi-agent-implementation-clarification-2026-05-18|多 Agent 代码实现澄清]]。

Workflow 与 TUI session 的边界要分清：Workflow 负责服务端编排，不会自动恢复某个 TUI session；TUI session 负责人机对话连续性。需要让 workflow 基于历史对话继续工作时，先通过 `/v1/sessions/{session_id}/turns` 读取 turns，人工或程序生成摘要，再放入 workflow `task.input`。更完整说明见 [[reference-legion-agent-session-001|会话连续性]]。

```
┌─────────────────────────────────────────┐
│              agent serve                │
│                                         │
│  HTTP API ──► Workflow Engine           │
│               ├─ parallel node          │
│               │   ├─► task A (goroutine)│
│               │   └─► task B (goroutine)│
│               └─ sequence node          │
│                   └─► task C            │
└─────────────────────────────────────────┘
```

### 启动 serve 模式

```powershell
agent serve --config agent.json
```

默认监听 `:8080`，可通过 `server.listen_addr` 或 `--addr` 修改。

---

### 节点类型速查

| 节点 `kind` | 作用 | 关键字段 |
|-------------|------|----------|
| `sequence` | 串行执行子节点，前一个完成才执行下一个 | `children` |
| `parallel` | **并发执行**所有子节点（goroutine 并行） | `children`, `failure_policy` |
| `agent_task` | 提交一个具体 task 给 Scheduler | `task.id`, `task.input` |
| `condition` | 按变量值走 then/else 分支 | `condition_key`, `condition_value`, `then`, `else` |
| `loop` | 循环执行 body，最多 N 次 | `loop_max_iterations`, `loop_body` |
| `join` | quorum 会合：N 个子节点中至少 M 个成功才继续 | `children`, `quorum` |
| `subworkflow` | 嵌套子工作流 | `subworkflow` |
| `wait_event` | 挂起等待外部事件（可通过 `/v1/workflows/{id}/events` 恢复） | `event_type`, `event_task_id` |
| `approval` | 挂起等待人工审批 | `subject`, `reason` |
| `error_handler` | try/catch 错误处理 | `try`, `handler` |

`parallel` 节点的 `failure_policy`：
- `fail_fast`（默认）：任一分支失败立即停止
- `collect_all`：收集所有分支结果再决定

---

### 示例

#### 两个 Task 并行执行

```json
POST /v1/workflows
Authorization: Bearer <admin-token>
Content-Type: application/json

{
  "id": "parallel-analysis",
  "root": {
    "id": "par",
    "kind": "parallel",
    "failure_policy": "collect_all",
    "children": [
      {
        "id": "task-market",
        "kind": "agent_task",
        "task": { "id": "t1", "input": "分析市场趋势" }
      },
      {
        "id": "task-competitor",
        "kind": "agent_task",
        "task": { "id": "t2", "input": "分析竞品动态" }
      }
    ]
  }
}
```

#### 并行后串行汇总

```json
{
  "id": "research-and-summarize",
  "root": {
    "id": "seq-root",
    "kind": "sequence",
    "children": [
      {
        "id": "research-phase",
        "kind": "parallel",
        "children": [
          { "id": "t-a", "kind": "agent_task", "task": { "id": "a1", "input": "调研 A 方向" } },
          { "id": "t-b", "kind": "agent_task", "task": { "id": "b1", "input": "调研 B 方向" } }
        ]
      },
      {
        "id": "summarize",
        "kind": "agent_task",
        "task": { "id": "s1", "input": "综合以上调研结果，输出结论报告" }
      }
    ]
  }
}
```

后续 `agent_task.task.input` 可以引用已经完成的 task result：

```json
{
  "id": "writer",
  "kind": "agent_task",
  "task": {
    "id": "writer-task",
    "agent_id": "writer",
    "input": "根据前序调研结果整理说明：{{tasks.research-task.result}}"
  }
}
```

占位符格式为 `{{tasks.<task_id>.result}}`。Workflow Engine 会从 `task_completed` 事件中读取对应 `task_id` 的最新 `message`，渲染到后续 task input。若对应结果尚不存在，占位符会保持原样，便于等待事件恢复或人工检查。

#### 条件分支

```json
{
  "id": "conditional-flow",
  "variables": { "mode": "fast" },
  "root": {
    "id": "cond",
    "kind": "condition",
    "condition_key": "mode",
    "condition_value": "fast",
    "then": {
      "id": "fast-task",
      "kind": "agent_task",
      "task": { "id": "f1", "input": "快速摘要" }
    },
    "else": {
      "id": "deep-task",
      "kind": "agent_task",
      "task": { "id": "d1", "input": "深度分析" }
    }
  }
}
```

#### 挂起等待人工审批后继续

```json
{
  "id": "approval-flow",
  "root": {
    "id": "seq",
    "kind": "sequence",
    "children": [
      { "id": "draft", "kind": "agent_task", "task": { "id": "dr1", "input": "生成草稿" } },
      { "id": "review", "kind": "approval", "subject": "draft-review", "reason": "需要人工确认草稿内容" },
      { "id": "publish", "kind": "agent_task", "task": { "id": "pub1", "input": "发布审批通过的草稿" } }
    ]
  }
}
```

审批通过后，向以下接口发送事件恢复执行：

```powershell
curl -X POST http://localhost:8080/v1/workflows/approval-flow/events \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"type": "approval_granted", "task_id": "draft-review", "message": "approved"}'
```

---

### 查询 Workflow 状态

```powershell
# 查询指定 workflow 结果
curl -H "Authorization: Bearer <token>" http://localhost:8080/v1/workflows/parallel-analysis

# 查询所有挂起等待的 workflow
curl -H "Authorization: Bearer <token>" http://localhost:8080/v1/workflows/waiting
```

### 从 session 历史启动 workflow

适用于“我在 TUI 中聊了一段，现在希望服务端 workflow 接着处理”的场景。

1. 查询目标 session：

```powershell
curl -H "Authorization: Bearer <token>" "http://localhost:8080/v1/sessions?company_id=cli-company&agent_id=cli-agent"
```

2. 查询最近 turns：

```powershell
curl -H "Authorization: Bearer <token>" "http://localhost:8080/v1/sessions/session-1770000000000000000/turns?limit=10"
```

3. 把 turns 摘要写入 workflow input：

```json
{
  "id": "wf-session-followup",
  "root": {
    "id": "seq",
    "kind": "sequence",
    "children": [
      {
        "id": "writer",
        "kind": "agent_task",
        "task": {
          "id": "writer-session-followup",
          "agent_id": "writer",
          "input": "基于 session-1770000000000000000 的对话摘要：<摘要>，继续写用户手册。"
        }
      }
    ]
  }
}
```

4. 如果 workflow 内后续节点要使用前序结果，用 `{{tasks.<task_id>.result}}`：

```json
{
  "id": "reviewer",
  "kind": "agent_task",
  "task": {
    "id": "review-session-followup",
    "agent_id": "reviewer",
    "input": "审查 writer 输出：{{tasks.writer-session-followup.result}}"
  }
}
```
<!-- @end-section -->

---

<!-- @section: multi-process -->
## 方式二：多进程 + 共享文件（手动并行）

适合人工驱动的并行探索场景：多个终端各自运行一个 `agent tui` 实例，通过共享文件系统交换中间结果。

```powershell
# 终端 1 – 用 coder profile 执行编码任务
agent tui --config agent.json --maas-profile coder

# 终端 2 – 用 reviewer profile 执行审查任务
agent tui --config agent.json --maas-profile reviewer
```

**共享手段**：
- 两个进程共享同一个 `agent.db`（SQLite），audit 和 task 记录可互见
- 用 `write_file` 工具写出中间结果，另一个 agent 用 `read_file` 读取
- 用 `search_content` 工具在 workspace 内检索前一个 agent 的输出

**注意**：SQLite 默认使用文件锁，并发写入有等待，通常不影响读取。
<!-- @end-section -->

---

<!-- @section: cross-process-routing -->
## 方式三：跨进程路由（待实现）

`TaskSpec` 中已保留 `agent_id` 字段，设计上用于将 task 路由到指定 agent 实例。当前 `agent serve` 已能在单进程内按 `agent_id` 选择子 Agent 配置、MaaS profile、上下文文件和工具 workspace；跨进程分发逻辑尚未实现。

```json
{
  "id": "t1",
  "agent_id": "agent-worker-2",
  "input": "处理子任务"
}
```

如需现在实现跨进程协作，可在外部用脚本或编排工具（如 PowerShell、Makefile、CI pipeline）分别调用各 agent 实例的 `/v1/tasks` HTTP API。
<!-- @end-section -->

---

<!-- @section: decision-guide -->
## 选择指南

```
需要有状态追踪 / 审批 / 挂起恢复？
  └─ YES → 方式一：agent serve + Workflow Engine

任务之间有数据依赖（A 的输出是 B 的输入）？
  └─ YES → 方式一：sequence 节点串联

任务完全独立，只需要结果汇总？
  └─ YES → 方式一：parallel 节点并发，collect_all 策略
  └─ 或   → 方式二：多终端手动并行

人工参与决策 / 审查？
  └─ YES → 方式一：approval 节点挂起 + /events 恢复

只是快速探索，不需要持久化追踪？
  └─ YES → 方式二：多终端 TUI
```
<!-- @end-section -->

## 相关文档

- [[reference-configuration-001|配置参考手册]] — `server`、`storage`、`runtime` 等配置字段
- [[reference-maas-model-profiles-001|MaaS Model Profiles]] — 多 profile 配置与 CLI 用法
