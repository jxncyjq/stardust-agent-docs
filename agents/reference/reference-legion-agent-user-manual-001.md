---
id: "reference-legion-agent-user-manual-001"
title: "Legion Agent 使用手册总览"
aliases: ["Legion Agent user manual", "Agent 使用手册", "TUI 使用手册", "agent tui"]
type: "reference"
category: "agents/reference"
tags: ["agent", "manual", "index", "tui", "cli", "session", "maas"]
version: "2.0.0"
created: "2026-05-19"
updated: "2026-08-27"
author: "jxncyjq"
status: "published"
parent: "reference-index-001"
children:
  - "reference-legion-agent-quick-start-001"
  - "reference-legion-agent-config-context-001"
  - "reference-legion-agent-tui-001"
  - "reference-legion-agent-session-001"
  - "reference-legion-agent-multi-agent-usage-001"
  - "reference-legion-agent-tasks-md-001"
  - "reference-legion-agent-tools-001"
  - "reference-legion-agent-cli-001"
  - "reference-legion-agent-http-service-001"
  - "reference-legion-agent-backend-api-001"
  - "reference-legion-agent-auth-001"
  - "reference-legion-agent-integration-001"
  - "reference-legion-gateway-001"
  - "reference-legion-agent-troubleshooting-001"
related_docs:
  - id: "reference-multi-agent-collaboration-001"
    relation: "related_to"
    path: "./multi-agent-collaboration.md"
  - id: "reference-maas-model-profiles-001"
    relation: "related_to"
    path: "../legion-agent/model-profiles.md"
  - id: "agent-configuration-001"
    relation: "related_to"
    path: "../legion-agent/configuration.md"
---

# Legion Agent 使用手册总览

Legion Agent 是 Agent Engine 的独立 Go 运行时，当前主要面向本地工程协作、TUI 对话、MaaS 推理调用、上下文文件加载、多 Agent 路由和 SQLite 持久化。

日常使用优先选择 `agent tui`。需要脚本化单次任务时使用 `agent run --plain`。需要 HTTP API、workflow、观测和运维接口时使用 `agent serve`。

## 模块索引

| 模块 | 文档 | 适用问题 |
|------|------|----------|
| 快速开始 | [[reference-legion-agent-quick-start-001\|快速开始]] | 项目在哪、怎么启动、怎么验证 |
| 配置与上下文 | [[reference-legion-agent-config-context-001\|配置与上下文文件]] | `agent.json`、MaaS、`AGENTS.md/SOUL.md/TOOLS.md/USER.md/MEMORY.md` |
| TUI | [[reference-legion-agent-tui-001\|TUI 使用]] | 交互界面、按键、slash 命令、显示配置 |
| 会话 | [[reference-legion-agent-session-001\|会话连续性]] | session、turn、上下文连续、会话切换 |
| 多 Agent | [[reference-legion-agent-multi-agent-usage-001\|多 Agent 调用]] | `@researcher`、`@writer`、子 Agent 配置 |
| 协作任务 | [[reference-legion-agent-tasks-md-001\|tasks.md 协作规范]] | 共享任务账本、Agent 交接、消息摘要和归档 |
| 工具能力 | [[reference-legion-agent-tools-001\|工具能力]] | 工具注册、schema、真实调用链路、文件/TaskLedger/AgentMessage 工具 |
| CLI | [[reference-legion-agent-cli-001\|CLI 命令速查]] | `run`、`serve`、`backup`、`data`、`skill` |
| HTTP 服务 | [[reference-legion-agent-http-service-001\|HTTP 服务]] | curl 速查：OpenAPI、SSE、治理、观测、task/workflow/session/message |
| 后端系统调用 | [[reference-legion-agent-backend-api-001\|后端系统调用参考]] | 端点全表、请求体、任务调用链、事件流、错误码 |
| 鉴权 | [[reference-legion-agent-auth-001\|鉴权与授权参考]] | admin token、loopback 握手、RBAC、工具授权、人工闸门 |
| 接入 | [[reference-legion-agent-integration-001\|接入参考]] | GUI 内嵌、本机自连、远端服务三种接入形态 |
| IM 网关 | [[reference-legion-gateway-001\|Legion Gateway IM 网关参考]] | `cmd/gateway`、平台适配、会话绑定、投递重试 |
| 排障 | [[reference-legion-agent-troubleshooting-001\|常见问题排查]] | TUI 不交互、模型未调用、颜色异常、上下文不连续 |

## 推荐阅读路径

首次使用：

1. [[reference-legion-agent-quick-start-001|快速开始]]
2. [[reference-legion-agent-config-context-001|配置与上下文文件]]
3. [[reference-legion-agent-tui-001|TUI 使用]]

需要多轮上下文和多 Agent：

1. [[reference-legion-agent-session-001|会话连续性]]
2. [[reference-legion-agent-multi-agent-usage-001|多 Agent 调用]]
3. [[reference-legion-agent-tasks-md-001|tasks.md 协作规范]]
4. [[reference-legion-agent-tools-001|工具能力]]
5. [[multi-agent-collaboration|多 Agent 协作参考手册]]

需要服务化 / 对接后端：

1. [[reference-legion-agent-http-service-001|HTTP 服务]]
2. [[reference-legion-agent-backend-api-001|后端系统调用参考]]
3. [[reference-legion-agent-auth-001|鉴权与授权参考]]
4. [[reference-legion-agent-integration-001|接入参考]]
5. [[reference-legion-gateway-001|Legion Gateway IM 网关参考]]
6. [[agent-operations-001|Legion Agent 运维 Runbook]]

## 一分钟选择指南

| 你现在想做什么 | 直接看 |
|----------------|--------|
| 第一次启动，确认能对话 | [[reference-legion-agent-quick-start-001\|快速开始]] |
| 配 DeepSeek/OpenAI-compatible/Legion MaaS | [[reference-legion-agent-config-context-001\|配置与上下文文件]] |
| 日常在终端里和 Agent 对话 | [[reference-legion-agent-tui-001\|TUI 使用]] |
| 让对话保持上下文 | [[reference-legion-agent-session-001\|会话连续性]] |
| 用 researcher/writer 分工 | [[reference-legion-agent-multi-agent-usage-001\|多 Agent 调用]] |
| 让多个 Agent 通过任务账本交接 | [[reference-legion-agent-tasks-md-001\|tasks.md 协作规范]] |
| 了解模型能调用哪些工具、为什么没有真实调用 | [[reference-legion-agent-tools-001\|工具能力]] |
| 在脚本或 CI 中执行一次任务 | [[reference-legion-agent-cli-001\|CLI 命令速查]] |
| 接入外部系统或 workflow | [[reference-legion-agent-integration-001\|接入参考]]、[[reference-legion-agent-backend-api-001\|后端系统调用参考]] |
| 配 token、RBAC、审批、插件授权 | [[reference-legion-agent-auth-001\|鉴权与授权参考]] |
| 把 Agent 接到 Telegram 之类 IM | [[reference-legion-gateway-001\|Legion Gateway IM 网关参考]] |
| 从历史 session 恢复对话 | [[reference-legion-agent-session-001\|会话连续性]] |
| 出现模型、TUI、session、消息问题 | [[reference-legion-agent-troubleshooting-001\|常见问题排查]] |

## 最小日常流程

```powershell
cd F:\source\stardust\Legion\legion\legionAgent
go run ./cmd/agent -- doctor
go run ./cmd/agent -- tui --config .\agent.json
```

进入 TUI 后：

```text
总结当前 Agent 的功能
@researcher 调研 session/cache 实现
@writer 根据 researcher 的结果整理成说明
/sessions
/event
```

## 相关文档

- [[reference-legion-agent-backend-api-001|后端系统调用参考]] — 端点全表与调用链
- [[reference-legion-agent-auth-001|鉴权与授权参考]] — 五层鉴权模型
- [[reference-legion-agent-integration-001|接入参考]] — 三种接入形态
- [[reference-legion-gateway-001|Legion Gateway IM 网关参考]] — IM 接入
- [[reference-legion-agent-tasks-md-001|tasks.md 协作规范]] — 多 Agent 文件态任务账本
- [[reference-legion-agent-tools-001|工具能力]] — 工具机制与执行管线
- [[multi-agent-collaboration|多 Agent 协作参考手册]] — workflow 与多 Agent 协作方式
- [[agent-configuration-001|Legion Agent 配置]] — 完整配置字段说明
- [[agent-persona-files-001|运行时上下文文件]] — `agents.md`/`SOUL.md`/`TOOLS.md`/`USER.md`/`MEMORY.md`
- [[reference-maas-model-profiles-001|MaaS Model Profiles]] — 模型 profile 选择规则
- [[agent-http-api-001|Legion Agent HTTP API]] — HTTP endpoint 说明
- [[agent-sqlite-schema-001|agent.db 数据结构]] — SQLite 表结构说明
