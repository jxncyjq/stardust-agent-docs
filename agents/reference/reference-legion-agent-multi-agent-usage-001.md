---
id: "reference-legion-agent-multi-agent-usage-001"
title: "Legion Agent 多 Agent 调用"
aliases: ["@researcher", "@writer", "子 Agent 调用"]
type: "reference"
category: "agents/reference"
tags: ["agent", "multi-agent", "tui", "routing"]
version: "2.0.0"
created: "2026-05-19"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: "reference-legion-agent-user-manual-001"
children: []
related_docs:
  - id: "reference-multi-agent-collaboration-001"
    relation: "related_to"
    path: "./multi-agent-collaboration.md"
  - id: "reference-legion-agent-tasks-md-001"
    relation: "related_to"
    path: "./reference-legion-agent-tasks-md-001.md"
---

# Legion Agent 多 Agent 调用

多 Agent 有四种入口：

| 入口 | 适合场景 |
|------|----------|
| TUI `@agent` | 人类手动委托，最快上手 |
| `delegate_task` 工具 | 模型自己把子任务派给其他 Agent（只有根编排者有这个工具） |
| TaskLedger `--task` | 多 Agent 围绕同一任务交接上下文 |
| HTTP / workflow | 服务端自动编排多个 Agent |

## 注册子 Agent

`agent.json` 的 `agents` 字段注册子 Agent：

```json
{
  "agents": {
    "researcher": "configs/agents/researcher.example.json",
    "writer": "configs/agents/writer.example.json"
  }
}
```

子 Agent 配置可以覆盖 role、MaaS profile、上下文文件、workspace 和 skills。未配置的共享项继承根配置。

skills 跟随角色：主 Agent 使用根配置 `skills.install_root`，`@researcher`、`@writer` 等子 Agent 优先使用自己的 `skills.install_root`。运行时只会把当前角色目录下匹配到的技能挂载到 prompt 中，避免 writer 意外继承 researcher 的技能说明。

推荐最少注册两个角色：

| Agent | 职责 |
|-------|------|
| `researcher` | 搜集事实、阅读代码、输出证据和结论 |
| `writer` | 整理结构、写文档、输出可读说明 |

更复杂时可以增加 `reviewer`、`planner`、`executor`，但不要一开始拆太细。先让两个角色跑通，比一次性设计完整组织更稳。

## TUI 中调用

在 TUI 中输入 `@` 会显示已注册 Agent。选择后可以直接委托：

```text
@researcher 调研一下当前实现
@writer 整理成说明
```

当前 `@agent` 调用会在同一 TUI 会话中记录实际 `agent_id` 和 `model_profile`，便于后续追踪。

建议提示格式：

```text
@researcher 调研 internal/sessioncache 和 session controller，列出事实、证据文件和风险
@writer 根据 researcher 的结果，写一份 docs/agents/reference/session-cache.md 说明
```

明确角色、目标、输出位置，会比一句“整理一下”稳定很多。

如果要让目标 Agent 消费自己的未读 inbox，可以显式加 `--inbox`：

```text
@writer --inbox 根据最新消息整理成说明
```

TUI 会把发给 `writer` 的 unread `AgentMessage` 注入为 `AgentMessage inbox context`。目标 Agent 成功运行后，这批消息会标记为 read；如果模型调用失败，消息保持 unread，方便下一次重试。

如果要让多个 Agent 围绕同一个 TaskLedger 任务连续协作，可以绑定任务 ID：

```text
@researcher --task TASK-20260523-001 调研当前实现
@writer --task TASK-20260523-001 根据 researcher 的结果整理说明
```

绑定后，TUI 会把 `tasks/TASK-*.md` 的任务详情作为 `TaskLedger task context` 注入目标 Agent prompt。Agent 返回结果后，运行时会向同一任务追加 `result.appended` 事件，并重建 `tasks.md` 与任务详情投影。

## TUI 消息

TUI 已支持最小 inbox/outbox 操作。发送消息：

```text
/send writer 请根据 researcher 的调研结果整理成说明
```

查看当前 Agent 的未读 inbox：

```text
/inbox
```

`/send` 会写入持久化 `AgentMessage`；`/inbox` 默认只显示发给当前 Agent 的 unread 消息。模型侧也可通过 `send_message` / `read_messages` 工具使用同一套消息表。人工协作的典型闭环是先 `/send writer ...`，再 `@writer --inbox ...` 让 writer 读取并处理消息。

## HTTP 消息 API

`agent serve` 模式下，外部系统可通过 HTTP 管理面发送和查询消息：

```http
POST /v1/agents/writer/messages
Authorization: Bearer <admin-token>
Content-Type: application/json

{
  "company_id": "company-1",
  "from": "researcher",
  "task_id": "TASK-20260525-001",
  "type": "result",
  "summary": "缓存实现调研完成",
  "artifact": "docs/research/cache.md"
}
```

查询 writer 的 unread 消息：

```http
GET /v1/agents/writer/messages?company_id=company-1&status=unread
Authorization: Bearer <admin-token>
```

读取后标记已读：

```http
GET /v1/agents/writer/messages?company_id=company-1&status=unread&mark_read=true
Authorization: Bearer <admin-token>
```

## 模型自主委派：delegate_task

根编排者（默认运行时）额外持有三个工具，子 Agent（worker）没有：

| 工具 | 说明 | 为什么不给 worker |
|------|------|-------------------|
| `delegate_task` | 把子任务派给其他 Agent，子角色 `leaf`（默认，不能再派）或 `orchestrator`（可嵌套至深度上限） | worker 再派 worker 会让委派树无界 |
| `session_search` | 跨会话检索历史 | 查询不带公司/Agent 过滤，会越过 worker 的沙箱与简报边界 |
| `moa_consult` | 多模型 MoA 咨询 | 会绕过该 worker 被指派的模型 profile 并放大成本 |

这个不对称是设计而非遗漏，有测试锁定。

除此之外，子 Agent 与主 Agent 的工具集相同——**包括写文件**（子 Agent 现在用的是读写工作区 registry，不再是只读）。要限制某个子 Agent，用它自己配置里的 `runtime.disabled_tools`。

`delegate_task` 是 Sensitive 工具：Manual 模式下要人工审批才会真的派出去。子任务完成会发 `subtask_completed` 事件。

## 查询可用 Agent

```powershell
curl -H "Authorization: Bearer change-me" "http://127.0.0.1:8080/v1/agents"
```

返回 `agents` 数组，就是配置 `agents` 映射的 key。默认 Agent 不在列表里——提交任务时把 `agent_id` 留空即可命中它。

## 当前边界

Agent 间 inbox/outbox 的数据模型、工具、TUI 人工入口、`@agent --inbox` 消息式 handoff、workflow result handoff、HTTP message API 与模型自主 `delegate_task` 都已可用。复杂协作仍优先通过 TaskLedger、同一 session 的上下文、人工提示和 Workflow `task.agent_id` 路由衔接——没有全局自动调度器。

## tasks.md 协作账本

多 Agent 需要交换上下文时，优先使用 TaskLedger 协作账本：Agent 通过工具追加 `tasks/events/*.jsonl` 事件，`tasks.md` 与 `tasks/TASK-*.md` 由 TaskLedger 重建为人类可读投影。

典型流程：

```text
researcher -> append handoff event，artifact 指向 docs/ 或 memory/ 证据
TaskLedger -> rebuild tasks.md 与 tasks/TASK-20260519-001.md
writer     -> 读取 TaskLedger Context 与证据链接
writer     -> append result / review event
```

`tasks.md` 只保留活跃任务摘要，长报告放入 `docs/`，长期经验放入 `memory/`，避免任务账本膨胀。普通 Agent 不直接覆盖 `tasks.md` 或 `tasks/TASK-*.md`。完整规范见 [[reference-legion-agent-tasks-md-001|tasks.md 协作规范]]。

需要 workflow 并发、审批、挂起恢复时，请查看 [[multi-agent-collaboration|多 Agent 协作参考手册]]。

## 推荐协作套路

### 调研到写作

```text
@researcher --task TASK-20260525-001 调研当前 TUI session 和 cache 实现，输出证据路径
@writer --task TASK-20260525-001 把调研结果整理成用户手册
```

### 调研后消息交接

```text
/send writer researcher 已完成调研，重点是 session cache 只缓存短期上下文窗口，不是长期 memory
@writer --inbox 写成用户可读说明
```

### 外部系统触发 writer

```powershell
curl -X POST "http://127.0.0.1:8080/v1/agents/writer/messages" `
  -H "Authorization: Bearer change-me" `
  -H "Content-Type: application/json" `
  -d "{\"company_id\":\"company-1\",\"from\":\"researcher\",\"type\":\"handoff\",\"summary\":\"调研完成，请整理说明\"}"
```

然后在 TUI 中：

```text
@writer --inbox 处理外部系统发来的未读消息
```

## 相关文档

- [[reference-legion-agent-tools-001|工具能力]] — `delegate_task` / `session_search` / `moa_consult` 与工具差异
- [[reference-legion-agent-tasks-md-001|tasks.md 协作规范]] — 共享任务账本协议
- [[multi-agent-collaboration|多 Agent 协作]] — workflow 并发与跨进程协作
- [[reference-legion-agent-backend-api-001|后端系统调用参考]] — `/v1/agents`、消息与任务端点
- [[reference-legion-agent-config-context-001|配置与上下文文件]] — 子 Agent 配置与 `disabled_tools`
