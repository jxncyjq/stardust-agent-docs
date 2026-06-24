---
id: "reference-index-001"
title: "Legion Agent 参考手册索引"
aliases: ["reference index", "参考索引", "手册目录"]
type: "reference"
category: "agents/reference"
tags: ["index", "reference", "manual"]
version: "1.5.0"
created: "2026-05-18"
updated: "2026-05-26"
author: "jxncyjq"
status: "published"
parent: null
children:
  - "reference-legion-agent-user-manual-001"
  - "reference-legion-agent-quick-start-001"
  - "reference-legion-agent-config-context-001"
  - "reference-legion-agent-tui-001"
  - "reference-legion-agent-session-001"
  - "reference-legion-agent-multi-agent-usage-001"
  - "reference-legion-agent-tasks-md-001"
  - "reference-legion-agent-tools-001"
  - "reference-legion-agent-cli-001"
  - "reference-legion-agent-http-service-001"
  - "reference-legion-agent-troubleshooting-001"
  - "multi-agent-collaboration"
  - "spec-multi-agent-implementation-clarification-2026-05-18"
---

# Legion Agent 参考手册索引

本目录收录各功能模块的独立参考手册，每篇文档聚焦单一主题，可独立查阅。

## 手册列表

| 文档 | 说明 | 标签 |
|------|------|------|
| [[reference-legion-agent-user-manual-001\|Legion Agent 使用手册总览]] | 使用手册模块入口和推荐阅读路径 | agent, manual, index |
| [[reference-legion-agent-quick-start-001\|快速开始]] | 项目目录、最小配置、第一次 TUI 对话、多 Agent 入门和本地验证 | quick-start, startup |
| [[reference-legion-agent-config-context-001\|配置与上下文文件]] | `agent.json`、MaaS profile、上下文文件、docs/memory、tasks、子 Agent 配置和分角色 skills | configuration, context |
| [[reference-legion-agent-tui-001\|TUI 使用]] | TUI 区域、按键、slash 命令、`@agent`、prompt/thinking 和颜色配置 | tui, commands |
| [[reference-legion-agent-session-001\|会话连续性]] | session/turn 持久化、session context cache、会话命令、HTTP session 查询 | session, sqlite, cache |
| [[reference-legion-agent-multi-agent-usage-001\|多 Agent 调用]] | 子 Agent 注册、TUI `@agent`、分角色 skills、`--task`、`--inbox`、HTTP 消息和协作套路 | multi-agent, routing |
| [[reference-legion-agent-tasks-md-001\|tasks.md 协作规范]] | 多 Agent 共享任务账本、消息交接、归档和膨胀控制 | tasks, handoff |
| [[reference-legion-agent-tools-001\|工具能力]] | 工具注册、schema、真实调用链路、文件工具、TaskLedger、AgentMessage 和权限说明 | tools, taskledger, message |
| [[reference-legion-agent-cli-001\|CLI 命令速查]] | `run`、`tui`、`serve`、`backup`、`restore`、`data`、`skill` 命令和示例 | cli, commands |
| [[reference-legion-agent-http-service-001\|HTTP 服务]] | `agent serve`、认证、OpenAPI、SSE、task、workflow、session、message、治理和观测接口 | http, api |
| [[reference-legion-agent-troubleshooting-001\|常见问题排查]] | TUI、MaaS、OpenAI-compatible 400、颜色、session、多 Agent、消息和日志排查 | troubleshooting |
| [[multi-agent-collaboration\|多 Agent 协作]] | Workflow Engine 并发节点、多进程共享文件、跨进程路由的三种协作方式与示例 | multi-agent, workflow, parallel |
| [[spec-multi-agent-implementation-clarification-2026-05-18\|多 Agent 代码实现澄清]] | 澄清多 Agent 从历史差距到当前 per-agent routing、TaskLedger、Message Bus 的实现边界 | multi-agent, scheduler, coordinator |

## 完整覆盖清单

| 文档 | 主要内容 |
|------|----------|
| [[reference-legion-agent-user-manual-001\|使用手册总览]] | 模块索引、推荐阅读路径、一分钟选择指南、最小日常流程、相关专项文档 |
| [[reference-legion-agent-quick-start-001\|快速开始]] | 项目目录、启动前检查、运行方式、第一次 TUI 对话、第一次多 Agent 调用、本地验证 |
| [[reference-legion-agent-config-context-001\|配置与上下文文件]] | `agent.json`、最小配置、MaaS、session cache、上下文文件、docs/memory、tasks、子 Agent、分角色 skills |
| [[reference-legion-agent-tui-001\|TUI 使用]] | 启动、界面区域、按键、输入模式、slash 命令、操作示例、Prompt/Thinking、颜色能力 |
| [[reference-legion-agent-session-001\|会话连续性]] | 工作方式、推荐配置、Session Context Cache、TUI 会话命令、从 session 恢复、Session 与 Workflow、多 Agent 会话、HTTP 查询 |
| [[reference-legion-agent-multi-agent-usage-001\|多 Agent 调用]] | 注册子 Agent、TUI `@agent`、分角色 skills、`--task`、`--inbox`、TUI 消息、HTTP 消息 API、当前边界、推荐协作套路 |
| [[reference-legion-agent-tasks-md-001\|tasks.md 协作规范]] | 定位、配置、投影/归档/膨胀控制、文件分层、并发写入、模板、状态协议、消息类型、交接规则、P21 Message Bus 关系 |
| [[reference-legion-agent-tools-001\|工具能力]] | 工具执行链路、Registry 执行管线、文件工具、主/子 Agent 工具差异、TaskLedger、AgentMessage、调用展示、排查、权限边界 |
| [[reference-legion-agent-cli-001\|CLI 命令速查]] | 命令表、源码运行模板、`run`、`tui`、`serve`、`backup/restore`、`data`、`skill`、覆盖参数 |
| [[reference-legion-agent-http-service-001\|HTTP 服务]] | 启动、常用接口、认证、RBAC、OpenAPI、Task、Workflow、Session、Agent 消息、SSE、治理查询、观测排障 |
| [[reference-legion-agent-troubleshooting-001\|常见问题排查]] | TUI 不交互、模型未调用、OpenAI 400、颜色异常、上下文不连续、子 Agent 不可用、消息未读、伪工具调用、日志、数据库 |
| [[multi-agent-collaboration\|多 Agent 协作]] | 协作边界、Workflow Engine、节点类型、workflow 示例、session 历史启动 workflow、多进程共享文件、跨进程路由、选择指南 |
| [[spec-multi-agent-implementation-clarification-2026-05-18\|多 Agent 代码实现澄清]] | 结论、三层能力边界、历史差距记录、历史落地顺序、阶段边界、最小成功标准、文档解释关系 |

## 按问题检索

| 我想要... | 查看 |
|-----------|------|
| 第一次启动 Agent | [[reference-legion-agent-quick-start-001\|快速开始]] |
| 配 DeepSeek/OpenAI-compatible 模型 | [[reference-legion-agent-config-context-001\|配置与上下文文件]]、[[../legion-agent/model-profiles\|MaaS Model Profiles]] |
| 在 TUI 里使用 `/` 或 `@` | [[reference-legion-agent-tui-001\|TUI 使用]]、[[reference-legion-agent-multi-agent-usage-001\|多 Agent 调用]] |
| 让多个 Agent 交换任务上下文 | [[reference-legion-agent-tasks-md-001\|tasks.md 协作规范]] |
| 了解模型能调用哪些工具、为什么没有真实调用 | [[reference-legion-agent-tools-001\|工具能力]] |
| 保持多轮对话上下文 | [[reference-legion-agent-session-001\|会话连续性]] |
| 理解 session cache 和 memory 的区别 | [[reference-legion-agent-session-001\|会话连续性]]、[[reference-legion-agent-config-context-001\|配置与上下文文件]] |
| 查询历史 session | [[reference-legion-agent-session-001\|会话连续性]]、[[reference-legion-agent-http-service-001\|HTTP 服务]] |
| 从历史 session 恢复 TUI 对话 | [[reference-legion-agent-session-001\|会话连续性]] |
| 让 workflow 基于 session 历史继续处理 | [[reference-legion-agent-session-001\|会话连续性]]、[[multi-agent-collaboration\|多 Agent 协作]] |
| 通过 HTTP 发送 Agent 消息 | [[reference-legion-agent-http-service-001\|HTTP 服务]]、[[reference-legion-agent-multi-agent-usage-001\|多 Agent 调用]] |
| 排查工具输出或伪工具调用 | [[reference-legion-agent-tools-001\|工具能力]]、[[reference-legion-agent-troubleshooting-001\|常见问题排查]] |
| `@writer --inbox` 没读到消息 | [[reference-legion-agent-troubleshooting-001\|常见问题排查]] |
| 备份、导出、清理数据 | [[reference-legion-agent-cli-001\|CLI 命令速查]] |
| 服务化接入 API | [[reference-legion-agent-http-service-001\|HTTP 服务]]、[[../legion-agent/http-api\|Legion Agent HTTP API]] |
| 排查模型没有调用 | [[reference-legion-agent-troubleshooting-001\|常见问题排查]] |

## 按功能主题检索

| 主题 | 入口 |
|------|------|
| 启动与运行 | [[reference-legion-agent-quick-start-001\|快速开始]]、[[reference-legion-agent-cli-001\|CLI 命令速查]]、[[reference-legion-agent-tui-001\|TUI 使用]] |
| 配置与模型 | [[reference-legion-agent-config-context-001\|配置与上下文文件]]、[[../legion-agent/model-profiles\|MaaS Model Profiles]] |
| 人格化上下文 | [[reference-legion-agent-config-context-001\|配置与上下文文件]]、[[../legion-agent/persona-files\|运行时上下文文件]] |
| TUI 操作 | [[reference-legion-agent-tui-001\|TUI 使用]]、[[reference-legion-agent-troubleshooting-001\|常见问题排查]] |
| 工具调用 | [[reference-legion-agent-tools-001\|工具能力]]、[[reference-legion-agent-troubleshooting-001\|常见问题排查]] |
| 多轮上下文 | [[reference-legion-agent-session-001\|会话连续性]]、[[reference-legion-agent-config-context-001\|配置与上下文文件]] |
| 多 Agent 路由 | [[reference-legion-agent-multi-agent-usage-001\|多 Agent 调用]]、[[spec-multi-agent-implementation-clarification-2026-05-18\|多 Agent 代码实现澄清]] |
| Agent 间消息 | [[reference-legion-agent-multi-agent-usage-001\|多 Agent 调用]]、[[reference-legion-agent-http-service-001\|HTTP 服务]] |
| tasks.md 协作 | [[reference-legion-agent-tasks-md-001\|tasks.md 协作规范]]、[[reference-legion-agent-tools-001\|工具能力]] |
| Workflow 编排 | [[multi-agent-collaboration\|多 Agent 协作]]、[[reference-legion-agent-http-service-001\|HTTP 服务]] |
| HTTP/API 接入 | [[reference-legion-agent-http-service-001\|HTTP 服务]]、[[../legion-agent/http-api\|Legion Agent HTTP API]] |
| 治理与观测 | [[reference-legion-agent-http-service-001\|HTTP 服务]]、[[../legion-agent/operations\|Legion Agent 运维 Runbook]] |
| 数据备份与保留 | [[reference-legion-agent-cli-001\|CLI 命令速查]]、[[../legion-agent/storage-ops\|存储运维]] |
| 实现边界与历史设计 | [[spec-multi-agent-implementation-clarification-2026-05-18\|多 Agent 代码实现澄清]]、[[multi-agent-collaboration\|多 Agent 协作]] |

## 相关目录

- [[../legion-agent/index\|legion-agent 文档索引]] — Legion Agent 全部设计、配置、运维文档入口
