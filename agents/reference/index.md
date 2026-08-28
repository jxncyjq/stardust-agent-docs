---
id: "reference-index-001"
title: "Legion Agent 参考手册索引"
aliases: ["reference index", "参考索引", "手册目录"]
type: "reference"
category: "agents/reference"
tags: ["index", "reference", "manual"]
version: "2.0.0"
created: "2026-05-18"
updated: "2026-08-28"
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
  - "reference-legion-agent-plugins-001"
  - "reference-legion-agent-cli-001"
  - "reference-legion-agent-http-service-001"
  - "reference-legion-agent-backend-api-001"
  - "reference-legion-agent-auth-001"
  - "reference-legion-agent-integration-001"
  - "reference-legion-gateway-001"
  - "reference-legion-agent-troubleshooting-001"
  - "reference-multi-agent-collaboration-001"
  - "spec-multi-agent-implementation-clarification-2026-05-18"
related_docs:
  - id: "agent-index-001"
    relation: "related_to"
    path: "../legion-agent/index.md"
---

# Legion Agent 参考手册索引

本目录收录各功能模块的独立参考手册，每篇聚焦单一主题，可独立查阅。

<!-- @section: usage-manuals -->
## 使用侧手册

| 文档 | 说明 | 标签 |
|------|------|------|
| [[reference-legion-agent-user-manual-001\|使用手册总览]] | 模块入口和推荐阅读路径 | agent, manual, index |
| [[reference-legion-agent-quick-start-001\|快速开始]] | 项目目录、最小配置、第一次 TUI 对话、本地验证 | quick-start, startup |
| [[reference-legion-agent-config-context-001\|配置与上下文文件]] | 全部配置块、MaaS profile、`agents.md` 分层、persona、环境变量 | configuration, context |
| [[reference-legion-agent-tui-001\|TUI 使用]] | 界面区域、按键、slash 命令（含 `/mode`、`/cwd`）、`@agent`、配色 | tui, commands |
| [[reference-legion-agent-session-001\|会话连续性]] | session/turn 持久化、上下文缓存、工作模式与工作目录、会话 HTTP 面 | session, sqlite, cache |
| [[reference-legion-agent-multi-agent-usage-001\|多 Agent 调用]] | 子 Agent 注册、`@agent`、`--task`、`--inbox`、`delegate_task` | multi-agent, routing |
| [[reference-legion-agent-tasks-md-001\|tasks.md 协作规范]] | 共享任务账本、消息交接、归档与膨胀控制 | tasks, handoff |
| [[reference-legion-agent-tools-001\|工具能力]] | lazy 元工具协议、工具循环、文件/任务/消息/网络/浏览器工具、模式与审批 | tools, browser, toolauth |
| [[reference-legion-agent-plugins-001\|WASM 插件手册]] | 插件 ABI、plugin.json、打包签名发布、install/grant/resolve、状态与收敛 | plugin, wasm, abi, signing |
| [[reference-legion-agent-cli-001\|CLI 命令速查]] | `run`、`tui`、`serve`、`plugins`、`backup`、`data`、`skill` | cli, commands |
| [[reference-legion-agent-troubleshooting-001\|常见问题排查]] | TUI、模型、工具、HTTP、token 膨胀、日志与数据库 | troubleshooting |

<!-- @end-section -->

<!-- @section: backend-manuals -->
## 后端 / 接入侧手册

| 文档 | 说明 | 标签 |
|------|------|------|
| [[reference-legion-agent-backend-api-001\|后端系统调用参考]] | 端点全表、请求体、任务调用链、事件流、错误码口径 | backend, http, api |
| [[reference-legion-agent-auth-001\|鉴权与授权参考]] | admin token、loopback 握手与 Origin 守卫、RBAC、工具授权、人工闸门 | auth, rbac, security |
| [[reference-legion-agent-integration-001\|接入参考]] | GUI 内嵌 / 本机自连 / 远端服务三种形态与最小调用序列 | integration, client |
| [[reference-legion-gateway-001\|Legion Gateway IM 网关参考]] | `cmd/gateway`：平台适配、会话绑定、PII 规则、投递重试 | gateway, im, telegram |
| [[reference-legion-agent-http-service-001\|HTTP 服务]] | 面向使用者的 curl 速查 | http, api |

<!-- @end-section -->

<!-- @section: design-notes -->
## 协作与设计说明

| 文档 | 说明 | 标签 |
|------|------|------|
| [[multi-agent-collaboration\|多 Agent 协作]] | Workflow Engine、`delegate_task`、多进程共享文件、跨进程路由现状 | multi-agent, workflow |
| [[spec-multi-agent-implementation-clarification-2026-05-18\|多 Agent 代码实现澄清]] | 从历史差距到当前 routing / TaskLedger / Message Bus 的实现边界 | multi-agent, scheduler |

<!-- @end-section -->

<!-- @section: by-question -->
## 按问题检索

| 我想要... | 查看 |
|-----------|------|
| 第一次启动 Agent | [[reference-legion-agent-quick-start-001\|快速开始]] |
| 配 DeepSeek / OpenAI-compatible 模型 | [[reference-legion-agent-config-context-001\|配置与上下文文件]]、[[reference-maas-model-profiles-001\|MaaS Model Profiles]] |
| 在 TUI 里用 `/`、`@`、`/mode`、`/cwd` | [[reference-legion-agent-tui-001\|TUI 使用]] |
| 让多个 Agent 交换任务上下文 | [[reference-legion-agent-tasks-md-001\|tasks.md 协作规范]]、[[reference-legion-agent-multi-agent-usage-001\|多 Agent 调用]] |
| 了解模型能调哪些工具、为什么没真实调用 | [[reference-legion-agent-tools-001\|工具能力]] |
| 写一个 WASM 插件 / 发布 / 装到部署里 | [[reference-legion-agent-plugins-001\|WASM 插件手册]] |
| 保持多轮上下文 / 查询历史 session | [[reference-legion-agent-session-001\|会话连续性]] |
| 知道后端到底有哪些接口 | [[reference-legion-agent-backend-api-001\|后端系统调用参考]] |
| 配 token、RBAC、审批、插件授权 | [[reference-legion-agent-auth-001\|鉴权与授权参考]] |
| 把 Agent 接进自己的系统 | [[reference-legion-agent-integration-001\|接入参考]] |
| 把 Agent 接到 Telegram 之类 IM | [[reference-legion-gateway-001\|Legion Gateway IM 网关参考]] |
| 中断一个跑飞的任务 | [[reference-legion-agent-backend-api-001\|后端系统调用参考]] |
| 拿到任务生成的文件 | [[reference-legion-agent-backend-api-001\|后端系统调用参考]]、[[reference-legion-agent-http-service-001\|HTTP 服务]] |
| 备份、导出、清理数据 | [[reference-legion-agent-cli-001\|CLI 命令速查]] |
| 排查伪工具调用、工具熔断、token 膨胀 | [[reference-legion-agent-troubleshooting-001\|常见问题排查]] |

<!-- @end-section -->

<!-- @section: by-topic -->
## 按功能主题检索

| 主题 | 入口 |
|------|------|
| 启动与运行 | [[reference-legion-agent-quick-start-001\|快速开始]]、[[reference-legion-agent-cli-001\|CLI 命令速查]]、[[reference-legion-agent-tui-001\|TUI 使用]] |
| 配置与模型 | [[reference-legion-agent-config-context-001\|配置与上下文文件]]、[[reference-maas-model-profiles-001\|MaaS Model Profiles]] |
| 人格化上下文 | [[reference-legion-agent-config-context-001\|配置与上下文文件]]、[[agent-persona-files-001\|运行时上下文文件]] |
| 工具与浏览器 | [[reference-legion-agent-tools-001\|工具能力]] |
| 插件（编写/发布/安装） | [[reference-legion-agent-plugins-001\|WASM 插件手册]] |
| 多轮上下文 | [[reference-legion-agent-session-001\|会话连续性]] |
| 多 Agent 路由 | [[reference-legion-agent-multi-agent-usage-001\|多 Agent 调用]]、[[multi-agent-collaboration\|多 Agent 协作]] |
| Agent 间消息 | [[reference-legion-agent-multi-agent-usage-001\|多 Agent 调用]]、[[reference-legion-agent-backend-api-001\|后端系统调用参考]] |
| tasks.md 协作 | [[reference-legion-agent-tasks-md-001\|tasks.md 协作规范]] |
| Workflow 编排 | [[multi-agent-collaboration\|多 Agent 协作]]、[[reference-legion-agent-backend-api-001\|后端系统调用参考]] |
| HTTP / API 接入 | [[reference-legion-agent-backend-api-001\|后端系统调用参考]]、[[reference-legion-agent-integration-001\|接入参考]]、[[agent-http-api-001\|Legion Agent HTTP API]] |
| 鉴权与安全 | [[reference-legion-agent-auth-001\|鉴权与授权参考]]、[[agent-security-tenancy-001\|安全与多租户]]、[[agent-governance-rbac-001\|Governance RBAC]] |
| IM 网关 | [[reference-legion-gateway-001\|Legion Gateway IM 网关参考]] |
| 治理与观测 | [[reference-legion-agent-backend-api-001\|后端系统调用参考]]、[[agent-operations-001\|Legion Agent 运维 Runbook]] |
| 数据备份与保留 | [[reference-legion-agent-cli-001\|CLI 命令速查]]、[[agent-storage-ops-001\|存储运维]] |

<!-- @end-section -->

## 相关文档

- [[agent-index-001|legion-agent 文档索引]] — Legion Agent 全部设计、配置、运维文档入口
- [[reference-legion-agent-user-manual-001|使用手册总览]] — 使用侧入口
- [[reference-legion-agent-backend-api-001|后端系统调用参考]] — 后端侧入口
