---
id: "design-agent-interop-protocols-001"
title: "Agent 互操作协议对比：A2A vs ACP vs MCP（Legion 落地成本）"
aliases: ["A2A vs ACP vs MCP", "agent interop", "互操作协议对比", "A2A ACP MCP"]
type: "design"
category: "design/architecture/agent_components"
tags: ["agent-interop", "a2a", "acp", "mcp", "protocol", "decision"]
version: "1.0.0"
created: "2026-07-30"
updated: "2026-07-30"
author: "jxncyjq"
status: "draft"
parent: null
children: []
related_docs:
  - id: "spec-agent-runtime-001"
    relation: "references"
    path: "./agent-runtime-spec.md"
  - id: "spec-agent-coordinator-002"
    relation: "references"
    path: "./agent-coordinator-spec.md"
  - id: "spec-agent-cognitive-core-000"
    relation: "related_to"
    path: "./cognitive-core-spec.md"
---

# Agent 互操作协议对比：A2A vs ACP vs MCP（Legion 落地成本）

<!-- @section: overview -->
## 概述

决策支持文档。回答「Legion 要不要 / 怎么接入 agent 互操作协议」，对比三个常被混为一谈但**目标层完全不同**的协议，并逐一给出 Legion 现状差距与落地成本。

**一句话区分（方向不同，别混）：**

| 协议 | 全称 | 连接方向 | 类比 |
|---|---|---|---|
| **A2A** | Agent2Agent（Google→Linux Foundation） | **agent ↔ agent**（跨厂商网状） | 不同公司的 agent 互相发现+协作 |
| **ACP** | Agent Client Protocol（Zed 等） | **客户端/编辑器 ↔ agent** | Zed/Cursor 驱动一个 agent 干活 |
| **MCP** | Model Context Protocol（Anthropic） | **agent ↔ 工具/数据** | 给 agent 挂工具，或把自己暴露成工具 |

**关键事实（本次调研坐实）：**
- Legion **三个都没实现**（内部编排是进程内 `delegate_task` 克隆 + Coordinator resolve，非任何标准互操作协议）。
- hermes-agent **只实现 ACP + MCP，不实现 A2A**（`acp_adapter/` = ACP 编辑器集成；`mcp_serve.py` = MCP 工具暴露）。→ 想抄 A2A 无参考；想抄 ACP/MCP hermes 是干净参考。
<!-- @end-section -->

<!-- @section: comparison -->
## 逐维度对比

| 维度 | A2A | ACP | MCP |
|---|---|---|---|
| **目的** | 跨厂商 agent 互操作/联邦 | 编辑器/客户端驱动 agent | 给 agent 供工具/数据；或把 agent 暴露成工具 |
| **传输** | HTTP(S) | stdio（也可 HTTP） | stdio / HTTP+SSE |
| **RPC** | JSON-RPC 2.0 | JSON-RPC 2.0 | JSON-RPC 2.0 |
| **发现** | **Agent Card** `/.well-known/agent-card.json`（url/skills/capabilities/securitySchemes） | `initialize` 握手声明 capabilities | `initialize` + `tools/list` |
| **核心方法** | `message/send`·`message/stream`·`tasks/get`·`tasks/cancel`·`tasks/pushNotificationConfig` | `initialize`·`new/load/resume/fork/list_sessions`·`cancel`·`prompt`(流)·`set_session_model/mode` | `initialize`·`tools/list`·`tools/call`·`resources/*`·`prompts/*` |
| **状态模型** | Task 生命周期(submitted/working/input-required/completed/failed/canceled) | Session + prompt turn（有 load/resume/fork） | 无长任务状态（工具调用为主，偏无状态） |
| **流式** | SSE：`TaskStatusUpdateEvent`/`TaskArtifactUpdateEvent` | 回调 `session_update`（message/thinking/step/tool_progress） | SSE / 通知 |
| **产出** | **Artifact** 对象 + **Message/Part**(text/file/data) | prompt 响应流 + 编辑提案(edit approval) | tool result（含结构化 content） |
| **认证** | OIDC / `securitySchemes`(Agent Card 声明) | `authenticate(method_id)` + auth_methods | 传输层(header/env)；无内建标准 |
| **权限/审批** | 无内建（靠 auth+opaque） | **内建**：allow_once/session/always/deny + 编辑审批 | 无内建（客户端侧决定） |
| **成熟度/生态** | 新(2024-25)，规范演进中，跨厂商愿景大、落地早期 | 编辑器侧(Zed 等)已用，SDK 稳 | 生态最成熟，Claude/Cursor/Codex 广泛支持 |
| **hermes 参考** | ❌ 无 | ✅ `acp_adapter/`（`HermesACPAgent(acp.Agent)`） | ✅ `mcp_serve.py` |
<!-- @end-section -->

<!-- @section: legion-fit -->
## Legion 现状差距（每协议缺什么）

### Legion 现有资产（可复用）
- HTTP 服务 + 自定义 REST 路由（`server/http.go`：`/v1/tasks`·`/v1/sessions`·`/v1/agents`·`/v1/events`）
- Task 生命周期（`domain.TaskStatus`：pending/assigned/running/quality_review/done/failed/suspended）
- Session 持久化 + 跨轮历史（sqlite）
- 事件流 `GET /v1/events`（token 流 + RuntimeEvent）
- 审批网关 `manualgate`（工具调用人工审批）
- Agent 注册表（AGENTS.md/registry，每 agent 有 role/model/tools/disabled_tools）

### A2A 差距（几乎全缺）
- ❌ Agent Card `/.well-known/agent-card.json`（`GET /v1/agents` 只返回名字 `[]string`，非 Card）
- ❌ JSON-RPC 2.0（现为 path+method REST）
- ❌ 标准方法名（`message/send` 等）
- ⚠️ Task 状态集不同（无 input-required/canceled）
- ❌ Message/Part/Artifact 标准对象（`task.Input` 是纯 string；`Artifact` 字段是内部 agent_message 自由文本，撞名不撞义）
- ❌ SSE 用 A2A 事件类型
- ❌ OIDC/securitySchemes

### ACP 差距（契合度最高）
- ⚠️ 需 JSON-RPC/stdio 适配层（内核已有 session/prompt/流式/审批，多为「包壳」）
- ✅ session new/load/resume/list ≈ Legion session store（已有）
- ✅ 流式 ≈ `/v1/events`（已有，需转 session_update 形状）
- ✅ 权限审批 ≈ `manualgate`（已有，需转 ACP allow_once/session/always/deny + 编辑提案）
- ✅ capabilities 声明 ≈ 现有工具集/模型

### MCP 差距（中等，独立）
- 需 stdio MCP server 把 Legion 的 sessions/messages/tasks 暴露成工具（`conversations_list`/`messages_send`/`events_poll` 式，仿 hermes `mcp_serve.py`）
<!-- @end-section -->

<!-- @section: cost -->
## 落地成本估算

| 协议 | 加什么 | 改内核? | hermes 参考 | Legion 契合 | 成本档 |
|---|---|---|---|---|---|
| **A2A** | 全新 gateway：Agent Card + JSON-RPC endpoint(6 方法) + 状态映射 + Message/Part/Artifact 包装 + SSE 事件类型 + OIDC | 否（旁挂 gateway） | ❌ 无 | 低（全新映射层） | **大**（L） |
| **ACP** | JSON-RPC/stdio adapter：initialize/session 生命周期/prompt 流/审批映射，多为包壳 | 否（adapter 复用 session/manualgate/events） | ✅ 干净参考(acp_adapter) | **高**（资产大多现成） | **中**（M） |
| **MCP** | stdio MCP server 暴露 sessions/tasks/messages 为工具 | 否（读现有 store） | ✅ `mcp_serve.py` | 中 | **中偏小**（S-M） |

> Go SDK：ACP 用 acp-go、MCP 用 mcp-go（`go.mod` 未见，需引入并核版本，本仓规矩：用前查版本/官方 doc，勿臆测签名）。A2A 无官方 Go SDK 时需照 spec 手写 JSON-RPC 层。
<!-- @end-section -->

<!-- @section: recommendation -->
## 推荐（决策树）

1. **想让外部编辑器/客户端（Zed/Cursor 类）驱动 Legion** → **ACP**。契合度最高、hermes 有干净参考、内核资产（session/流式/审批）大多现成，性价比最高。**首选。**
2. **想把 Legion 的会话/任务暴露给别的 agent 当工具用** → **MCP**。生态最成熟、独立、成本小。可与 ACP 并存。
3. **真要跨厂商 agent 联邦（别人的 agent 经网络发现并调用 Legion）** → **A2A**。成本最大、无 hermes 参考、生态早期。**除非有明确跨组织互操作需求，否则暂缓**；真做就旁挂 A2A gateway 不动内核。

**不互斥**：可对内客户端走 ACP、对外工具走 MCP、需要时再加 A2A 对外 agent。建议**先 ACP（或 MCP）验证一条互操作链**，A2A 待有真实跨厂商场景再上。

**下一步选项**：①起 ACP adapter 移植计划（抄 hermes）②起 MCP server 计划 ③起 A2A gateway 设计+计划。
<!-- @end-section -->

## 相关文档

- [[spec-agent-runtime-001|AgentRuntime 组件规范]] — Task 生命周期 / 工具执行（互操作层映射的内核）
- [[spec-agent-coordinator-002|AgentCoordinator 组件规范]] — 任务路由 / 多 agent 调度
- [[spec-agent-cognitive-core-000|CognitiveCore 组件规范]] — 上下文/会话（ACP session 映射相关）
