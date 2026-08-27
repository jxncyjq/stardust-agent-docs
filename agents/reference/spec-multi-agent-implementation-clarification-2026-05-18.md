---
id: "spec-multi-agent-implementation-clarification-2026-05-18"
title: "多 Agent 代码实现澄清"
aliases: ["multi-agent implementation clarification", "多 Agent 实现边界", "per-agent runtime routing"]
type: "spec"
category: "agents/specs"
tags: ["multi-agent", "workflow", "scheduler", "coordinator", "agent-registry"]
version: "1.2.0"
created: "2026-05-18"
updated: "2026-08-27"
author: "codex"
status: "published"
related_docs:
  - id: "spec-multi-agent-design-2026-05-18"
    relation: "clarifies"
    path: "../specs/2026-05-18-multi-agent-design.md"
  - id: "reference-multi-agent-collaboration-001"
    relation: "clarifies"
    path: "./multi-agent-collaboration.md"
---

# 多 Agent 代码实现澄清

## 结论

> 注：本文最初用于澄清 2026-05-18 的多 Agent 设计差距。P20、P21 和 P22 已在后续批次完成 runtime routing、TaskLedger、Message Bus 与 session continuity；当前剩余缺口主要是后续 L4 跨进程/跨节点 Agent。

当前 `legion/legionAgent` 代码已经具备 **多 task 工作流编排的基础结构**，并已通过 P20 补齐 **per-agent runtime routing**。

也就是说：

- 已有：Workflow DSL 的 `parallel` / `sequence` / `agent_task` 等节点结构。
- 已有：`TaskSpec.agent_id` 字段，能在工作流定义中表达“这个 task 希望由哪个 agent 执行”。
- 已有：单 Agent Runtime，可加载一套上下文文件、一个 MaaS client、一个 ToolRoot。
- 已完成：P20 已补齐 `TaskSpec.agent_id` 到不同 Agent 配置、人格、记忆、模型 profile、workspace 的运行时路由。
- 已完成：P20 已补齐 `agent serve` 默认启动链路中的 `workflow.Engine` 与后台 `Coordinator` 执行闭环。
- 已完成：P22 已补齐 session/turn 持久化、最近 N 轮上下文注入和跨 Agent 会话追踪。
- 已完成：P21A TaskLedger 文件态协作层。
- 已完成：P21B/P21C Agent 间 inbox/outbox 基础数据模型、工具、TUI `/send` / `/inbox`、`@agent --inbox` 消息式 handoff、workflow result handoff 和 HTTP message API。

因此，当前实现可以描述为“单进程多 Agent runtime routing 和本地协作闭环已完成”，但还不能描述为“跨进程/跨节点 Agent 网络已完成”。更准确的表述是：

> Workflow 编排模型、`agent_id` 路由、per-agent Runtime 构建、session continuity、TaskLedger 文件态协作、SQLite message bus 基础层、`@agent --inbox` 消息式 handoff、workflow result 传递和外部 message API 已具备。

## 三层能力边界

| 层级 | 名称 | 当前状态 | 说明 |
|------|------|----------|------|
| L1 | 多 task 编排 | 已完成 | `internal/workflow` 已支持 sequence/parallel/condition/loop/join/subworkflow/approval/wait_event/error_handler 等节点。 |
| L2 | per-agent runtime routing | 已完成 | 同一进程内根据 `task.AgentID` 切换 SOUL/MEMORY/TOOLS/USER、MaaS profile、ToolRoot、workspace。 |
| L3 | 组织化协作 | 已完成 | session continuity、TaskLedger、AgentMessage 存储/工具、TUI `/send` `/inbox`、`@agent --inbox`、workflow result handoff、message HTTP API 和 P21 compat golden 已完成。 |
| L4 | 跨进程/跨节点 Agent | 设计阶段 | 多个 Agent 进程或远程 runtime 通过 Adapter/消息总线协作。 |

P21 已完成本地/单进程多 Agent 协作闭环。L4 跨进程/跨节点不应混入 P21 批次，应作为后续独立阶段规划。

## 历史差距记录

以下内容记录 P20/P21/P22 完成前的实现差距，用于解释为什么需要 per-agent runtime routing、TaskLedger、AgentMessage 和 session continuity。当前代码已经完成这些修正，实际使用方式以本文结论、[[multi-agent-collaboration]] 和各 reference 手册为准。

### 1. Workflow agent_id 路由意图

`internal/workflow/definition.go` 中 `TaskSpec` 已有：

```go
type TaskSpec struct {
    ID        string `json:"id"`
    CompanyID string `json:"company_id,omitempty"`
    AgentID   string `json:"agent_id,omitempty"`
    Input     string `json:"input"`
}
```

`internal/workflow/engine.go` 的 `executeAgentTask` 会把 `node.Task.AgentID` 写入 `domain.Task.AgentID` 后加入 Scheduler。

历史问题：P20 前这只是“路由意图入队”，尚未真正进入指定 agent 的运行时。

当前状态：P20 后该字段会经 Scheduler 和 Coordinator 保留，并路由到对应 Agent runtime。

### 2. Scheduler 覆盖 AgentID

历史问题：`internal/task/scheduler.go` 的 `Next(ctx, agentID)` 曾经会直接执行：

```go
task.AgentID = agentID
```

这会覆盖 workflow 中已经写入的 `TaskSpec.agent_id`，导致 `agent_task.task.agent_id = "researcher"` 最终变成当前 Coordinator 固定 agent 的 ID。

当前正确行为为：

```go
if task.AgentID == "" {
    task.AgentID = agentID
}
```

也就是：已有 `Task.AgentID` 时保留；为空时才填默认 agent。

### 3. Coordinator 固定使用一个 Agent 和一个 Runtime

历史问题：`internal/runtime/coordinator.go` 曾经只持有一个 Agent 和一个 Runtime：

```go
type Coordinator struct {
    agent   domain.Agent
    runtime *Runtime
}
```

`Heartbeat()` 中执行：

```go
taskToRun, ok, err := c.scheduler.Next(ctx, c.agent.ID)
run, err := c.runtime.RunTask(ctx, c.agent, taskToRun)
```

这意味着即使 task 保留了 `AgentID=researcher`，执行时仍会使用 `c.agent` 和 `c.runtime`。

当前状态：Coordinator 会根据 `taskToRun.AgentID` 查询 AgentRegistry，临时构建或选择对应 runtime：

```text
task.AgentID
  -> AgentRegistry.Get(agentID)
  -> build context prefix from that agent config
  -> maas client from maas_profile
  -> tool root from context_files.root
  -> runtime.RunTask(agentDomain, task)
```

### 4. Config 缺少 agents 字段

历史问题：`internal/config/config.go` 的 `Config` 曾经没有：

```go
Agents map[string]string `json:"agents"`
```

所以主配置无法声明：

```json
{
  "agents": {
    "researcher": "agents/researcher.json",
    "writer": "agents/writer.json"
  }
}
```

当前状态：主配置已经支持 `agents` map，并通过 AgentRegistry 加载子 agent 配置。

### 5. `agent serve` 缺少 workflow/runtime 执行闭环注入

历史问题：`internal/server/http.go` 已经有 `/v1/workflows` handler，并且测试可以手动注入 `workflow.Engine`。

但 `internal/cli/command.go` 的 `newServeCommand` 曾经只把 `Tasks`、`Workflows` 等 store 传入 `server.NewHTTPServer`，没有构建并注入：

- `workflow.Engine`
- `workflow.EventBus`
- `Coordinator`
- AgentRegistry
- 后台 heartbeat job

当前状态：`agent serve` 已完成 workflow engine、workflow event bus、AgentRegistry、Coordinator 和后台 heartbeat 链路注入，可通过 HTTP 提交 workflow 并由后台执行。

## 历史落地顺序

以下 M1-M5 是 P20 规划时的落地顺序，当前均已完成，保留用于追踪设计意图。

### M1: 保留 TaskSpec.agent_id

修改 `Scheduler.Next()`：只在 task 没有 AgentID 时填入当前 coordinator agent。

验收：

- 原有无 AgentID task 仍会被默认 agent 领取。
- 已指定 AgentID 的 task 不会被覆盖。

### M2: 引入 AgentRegistry

新增 `internal/agentregistry`：

- 加载主配置 `agents` map。
- 解析子 agent JSON。
- 输出 `AgentConfig`。
- 支持 `Get(name)`、`Names()`。

子 agent 配置只描述 agent 差异项：

- `id`
- `role`
- `maas_profile`
- `context_files`
- `workspace`
- `skills`

主配置继续提供共享项：

- `maas.base_url` / `maas.api_key`
- `maas.profiles`
- `storage`
- `server`
- `runtime.max_tool_rounds`

### M3: Coordinator 支持 per-agent Runtime

Coordinator 保留默认 runtime 作为 fallback，但在 task 有 AgentID 且 registry 命中时，使用子 agent 配置构建 runtime。

必须保持向后兼容：

- `AgentRegistry == nil` 时行为不变。
- `task.AgentID == ""` 时行为不变。
- `task.AgentID` 未注册时 fallback 默认 runtime，并记录 warning event/audit。

### M4: serve 注入完整闭环

`agent serve` 启动时应完成：

1. 读取主配置。
2. 加载 AgentRegistry。
3. 构建 Scheduler / LockStore / Audit / Events / WorkflowEngine。
4. 构建 Coordinator。
5. 将 Coordinator 注册为后台 heartbeat job。
6. 将 WorkflowEngine 与 WorkflowEvents 注入 HTTPServer。

验收：

- `POST /v1/workflows` 能创建 task。
- 后台 heartbeat 能执行 task。
- 指定 `agent_id=researcher` 的 task 使用 researcher 的 SOUL/MEMORY/model profile。

### M5: 更新文档与配置样例

需要同步：

- `configs/agent.full.example.json` 新增 `agents` 示例。
- 新增 `configs/agents/researcher.example.json`、`writer.example.json`。
- 更新 `docs/agents/legion-agent/configuration.md`。
- 更新 `docs/agents/reference/multi-agent-collaboration.md` 中的实现状态。

## 不应在本阶段实现的内容

以下内容是 P20 规划时刻刻意排除的后续范围。当前 P21 已补齐本地消息总线，剩余仍属于后续 L4 或企业组织化治理：

- 完整组织树：Company / Department / Team / report chain。
- 五类通讯协议：delegate/report/mention/broadcast/data_pipeline。
- 跨进程 task 分发。
- `delegate_task` TUI 工具。
- Agent 招聘、预算、模型升级等完整治理门控。
- Agent 内部信用/拍卖经济层。

## 最小成功标准

用一个 workflow 验证：

```json
{
  "id": "multi-agent-smoke",
  "root": {
    "id": "parallel-root",
    "kind": "parallel",
    "failure_policy": "collect_all",
    "children": [
      {
        "id": "research-task",
        "kind": "agent_task",
        "task": {
          "id": "research-1",
          "agent_id": "researcher",
          "input": "调研多 Agent 实现现状"
        }
      },
      {
        "id": "writer-task",
        "kind": "agent_task",
        "task": {
          "id": "writer-1",
          "agent_id": "writer",
          "input": "整理调研结论"
        }
      }
    ]
  }
}
```

验收结果：

- Scheduler 中两个 task 保留各自 AgentID。
- Coordinator 执行 `research-1` 时加载 researcher 配置。
- Coordinator 执行 `writer-1` 时加载 writer 配置。
- Audit/Event 中能看到实际执行 AgentID。
- 未注册 agent_id 有明确 fallback 事件。

## 对两个文档的解释关系

`multi-agent-collaboration.md` 是面向使用者的协作方式说明，强调“怎么协作”。

`2026-05-18-multi-agent-design.md` 是面向实现者的修正设计，强调“历史上为什么还不是真多 Agent，以及如何让 agent_id 真的生效”。

当前编码应以后者为准：先补齐 per-agent runtime routing，再回头扩展组织化协作。

## 相关文档

- [[multi-agent-collaboration|多 Agent 协作]] — 当前可用的协作方式
- [[reference-legion-agent-multi-agent-usage-001|多 Agent 调用]] — 使用入口
- [[reference-legion-agent-backend-api-001|后端系统调用参考]] — 任务/workflow 端点现状
