---
id: "analysis-hermes-overview-001"
title: "Hermes Agent 项目架构总览"
aliases: ["hermes-agent overview", "Hermes Agent架构概览"]
type: "analysis"
category: "design/analysis/hermes"
tags: ["hermes-agent", "architecture", "python", "agent", "llm"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: null
children:
  - "analysis-hermes-runtime-002"
  - "analysis-hermes-tools-003"
  - "analysis-hermes-gateway-004"
  - "analysis-hermes-datamodels-005"
  - "analysis-hermes-insights-006"
  - "analysis-hermes-vs-evolver-007"
related_docs:
  - id: "analysis-hermes-runtime-002"
    relation: "related_to"
    path: "./02-agent-runtime.md"
  - id: "analysis-hermes-tools-003"
    relation: "related_to"
    path: "./03-tools-skills-plugins.md"
  - id: "analysis-hermes-gateway-004"
    relation: "related_to"
    path: "./04-gateway-cli-deployment.md"
  - id: "analysis-hermes-datamodels-005"
    relation: "related_to"
    path: "./05-data-models.md"
  - id: "analysis-hermes-insights-006"
    relation: "related_to"
    path: "./06-hermes-insights.md"
  - id: "analysis-hermes-vs-evolver-007"
    relation: "related_to"
    path: "./07-hermes-vs-evolver.md"
---

<!-- @section: overview -->
# Hermes Agent 项目架构总览

## 项目概述

Hermes Agent 是由 **Nous Research** 构建的"自我完善的 AI Agent"框架，是一个大型、社区驱动、支持多模态和多提供商的 AI Agent 平台。它不仅仅是单个 Agent，而是一个完整的 Agent 生态系统，包含多平台消息网关、终端 UI、Web 仪表盘、技能市场和插件系统。

- 仓库地址：`NousResearch/hermes-agent`
- 版本：v0.12.0（2026.04.30）
- 主要语言：Python 3.11+（后端/CLI）/ TypeScript + React 19（前端）
- 框架：Rich + prompt_toolkit（CLI），Vite + React 19（Web），Ink v6（TUI），FastAPI（Web Server）
- 状态存储：SQLite + WAL + FTS5
- 包管理：uv（Python），npm（Node.js）

## 系统定位

```
外部用户 / 消息平台
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│                   Hermes Agent 平台                       │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ CLI/TUI  │  │  Gateway │  │   ACP    │  入口层        │
│  │ (REPL)   │  │(19 平台) │  │(编辑器)  │               │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘               │
│       └──────────────┼─────────────┘                     │
│                      ▼                                    │
│  ┌──────────────────────────────────────────┐            │
│  │           AIAgent (核心引擎)              │            │
│  │                                          │            │
│  │  Prompt → LLM → Tool Calls → Loop        │            │
│  │     ↓          ↓            ↓            │            │
│  │  Transports  Memory   Context Compressor  │            │
│  └──────────────────────────────────────────┘            │
│                      │                                    │
│       ┌──────────────┼──────────────┐                    │
│       ▼              ▼              ▼                    │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐               │
│  │  Tools  │  │  Skills  │  │ Plugins  │  能力层        │
│  │(~68 工具)│  │(89 技能) │  │(可扩展) │               │
│  └─────────┘  └──────────┘  └──────────┘               │
│                      │                                    │
│                      ▼                                    │
│  ┌──────────────────────────────────────────┐            │
│  │     SessionDB (SQLite + FTS5)            │  持久层     │
│  └──────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────┘
```

## 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| 后端 | Python 3.11+, asyncio | Agent 引擎、网关、CLI |
| 前端 Web | React 19, TypeScript, Vite, Tailwind CSS v4 | 管理仪表盘 |
| 前端 TUI | React 19, Ink v6, TypeScript | 终端 UI |
| 网站 | Docusaurus 3.9 (React 19, MDX) | 文档站点 |
| 数据库 | SQLite (WAL 模式, FTS5) | 会话、消息、全文搜索 |
| LLM 客户端 | openai, anthropic SDK | 多提供商支持 |
| CLI 框架 | prompt_toolkit, Rich, argparse | 交互式 REPL |
| Web 服务 | FastAPI + WebSocket | 仪表盘后端 |
| 容器化 | Docker + docker-compose | 网关 + 仪表盘部署 |

## 分层架构

```
入口层 (Entry Points)
  ├── hermes_cli/       — CLI (30+ 子命令)
  ├── gateway/          — 多平台消息网关 (19 个平台适配器，含 helper/rate-limit 子模块共 30+ .py)
  ├── tui_gateway/      — 终端 UI JSON-RPC 后端
  ├── acp_adapter/      — Agent Client Protocol 服务器
  └── web/              — Web 仪表盘 (React SPA)
       │
       ▼
核心引擎层 (Core Engine)
  ├── run_agent.py      — AIAgent 类 (主循环, 14,123 行)
  ├── agent/            — Agent 内部模块 (56 个 .py 文件)
  │   ├── transports/   — 多提供商响应规范化
  │   ├── anthropic_adapter.py    — Claude 适配
  │   ├── bedrock_adapter.py      — AWS Bedrock
  │   ├── codex_responses_adapter.py — Codex/Responses API
  │   └── gemini_native_adapter.py   — Gemini 适配
  ├── model_tools.py    — 工具编排层
  └── toolsets.py       — 工具集定义与组合
       │
       ▼
能力层 (Capabilities)
  ├── tools/            — 86 个 .py 文件，~68 个 registry.register() 静态注册的工具（不含 MCP 动态发现）
  ├── skills/           — 89 个 SKILL.md 内置技能（25 个分类目录）
  ├── optional-skills/  — 60 个 SKILL.md 可选技能（15 个分类目录）
  └── plugins/          — 可插拔扩展系统（48 个 .py 文件）
       │
       ▼
持久层 (Persistence)
  ├── hermes_state.py   — SQLite SessionDB (FTS5)
  ├── hermes_constants.py — 路径与常量
  └── hermes_logging.py — 分级日志系统
```

## 核心数据流

```
用户输入 (CLI / Telegram / Discord / ACP...)
  │
  ▼
AIAgent.run_conversation()
  │
  ├── 1. 构建系统提示 (prompt_builder.py)
  │      ├── Agent 身份 + 平台提示
  │      ├── 技能索引注入
  │      ├── 上下文文件 (.hermes.md / SOUL.md)
  │      └── 模型特定执行指南
  │
  ├── 2. 初始化 Transport 层
  │      ├── AnthropicTransport / ChatCompletionsTransport
  │      ├── 消息格式转换 (OpenAI ↔ 各提供商)
  │      └── 响应规范化 → NormalizedResponse
  │
  ├── 3. API 调用循环 (while iterations < max)
  │      ├── 构建 API kwargs (tools, messages, system)
  │      ├── 调用 LLM (支持流式)
  │      ├── 错误分类 → 压缩/故障转移/重试
  │      └── 解析 NormalizedResponse
  │
  ├── 4. 工具调用执行
  │      ├── 看门狗检查 (tool_guardrails.py)
  │      ├── 并发执行 (ThreadPoolExecutor)
  │      ├── handle_function_call() → registry.dispatch()
  │      └── 结果追加到消息历史
  │
  ├── 5. 上下文管理
  │      ├── 压缩触发检查
  │      ├── 辅助模型摘要中间回合
  │      └── 注入预取记忆上下文
  │
  └── 6. 持久化 → SessionDB (SQLite + FTS5)
```

## 入口点对比

| 入口 | 命令 | 用途 |
|------|------|------|
| CLI | `hermes` / `hermes chat` | 交互式终端 REPL |
| TUI | `hermes --tui` | 基于 React/Ink 的终端 UI |
| Agent | `hermes-agent` | 单次 Agent 运行 |
| Gateway | `hermes gateway run` | 多平台消息网关 |
| Dashboard | `hermes dashboard` | Web 管理仪表盘 |
| ACP | `hermes-acp` | 编辑器集成 (VS Code/Zed/JetBrains) |
| MCP Server | `hermes mcp serve` | 作为 MCP 工具暴露 |

## 关键设计原则

1. **传输规范化**: 所有提供商响应规范化为 `NormalizedResponse`，避免提供商特定分支
2. **工具自注册**: 工具通过顶层 `registry.register()` 在导入时声明自身，使用 AST 扫描自动发现
3. **渐进式技能加载**: 技能元数据先加载（低成本），完整内容按需获取
4. **配置文件隔离**: 多个独立的 `HERMES_HOME` 目录，支持多配置文件
5. **提示缓存稳定性**: 对话中期永不修改系统提示/工具集，避免破坏 Anthropic 提示缓存
6. **插件非侵入性**: 插件绝不能修改核心文件，通过通用插件表面扩展

## 文档索引

- [[02-agent-runtime|Agent 运行时引擎分析]]
- [[03-tools-skills-plugins|工具、技能与插件系统分析]]
- [[04-gateway-cli-deployment|网关、CLI 与部署分析]]
- [[05-data-models|状态持久化与数据模型分析]]
- [[06-hermes-insights|Hermes 洞察与 Legion 参考]]

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[02-agent-runtime|Agent 运行时引擎分析]]
- [[03-tools-skills-plugins|工具、技能与插件系统分析]]
- [[04-gateway-cli-deployment|网关、CLI 与部署分析]]
- [[05-data-models|状态持久化与数据模型分析]]
- [[06-hermes-insights|Hermes 洞察与 Legion 参考]]

<!-- @end-section -->
