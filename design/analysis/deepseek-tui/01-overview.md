---
id: "analysis-deepseek-tui-overview-001"
title: "DeepSeek-TUI 项目总览"
aliases: ["deepseek-tui overview", "DeepSeek-TUI架构总览"]
type: "analysis"
category: "design/analysis/deepseek-tui"
tags: ["deepseek-tui", "architecture", "rust", "tui", "overview"]
version: "1.0.0"
created: "2026-05-07"
updated: "2026-05-07"
author: "jxncyjq"
status: "review"
parent: null
children:
  - "analysis-deepseek-tui-crates-002"
  - "analysis-deepseek-tui-tui-003"
  - "analysis-deepseek-tui-api-004"
  - "analysis-deepseek-tui-tools-005"
  - "analysis-deepseek-tui-storage-006"
  - "analysis-deepseek-tui-insights-007"
related_docs:
  - id: "analysis-deepseek-tui-crates-002"
    relation: "related_to"
    path: "./02-crate-analysis.md"
  - id: "analysis-deepseek-tui-tui-003"
    relation: "related_to"
    path: "./03-tui-components.md"
  - id: "analysis-deepseek-tui-api-004"
    relation: "related_to"
    path: "./04-api-client.md"
  - id: "analysis-deepseek-tui-tools-005"
    relation: "related_to"
    path: "./05-tool-system.md"
  - id: "analysis-deepseek-tui-storage-006"
    relation: "related_to"
    path: "./06-session-storage.md"
  - id: "analysis-deepseek-tui-insights-007"
    relation: "related_to"
    path: "./07-insights.md"
---

<!-- @section: overview -->

# DeepSeek-TUI 项目总览

## 项目概述

DeepSeek-TUI 是一个**基于终端的 LLM 交互客户端**，完全用 Rust 编写，支持 DeepSeek、OpenAI、OpenRouter 等 7 家 AI 提供商。它提供类似 Claude Code 的 Agent 工作模式（`Agent / Yolo / Plan`），内置工具执行引擎、会话持久化、MCP 集成和子代理系统。

- 版本：v0.8.11
- 主要语言：Rust（edition 2024）
- 工作区：14 个 crate，230+ Rust 源文件
- TUI 框架：Ratatui v0.29 + Crossterm v0.28

## 系统定位

```
终端用户
    │  键盘输入 / 命令
    ▼
┌──────────────────────────────────────────────────┐
│              DeepSeek-TUI (TUI 层)                │
│                                                  │
│  Onboarding → Agent/Yolo/Plan 模式                │
│  Composer → Transcript → Sidebar → Footer        │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │           核心引擎 (core crate)           │    │
│  │  Session → Turn → ToolExecution → Quota  │    │
│  └──────────────────────────────────────────┘    │
│                                                  │
│  ┌────────┐ ┌─────────┐ ┌──────┐ ┌───────────┐  │
│  │ shell  │ │  file   │ │ git  │ │  subagent │  │
│  └────────┘ └─────────┘ └──────┘ └───────────┘  │
│         ↓                                        │
│  MCP Servers / LSP / RLM (Python REPL)           │
└──────────────────────────────────────────────────┘
    │
    ▼
DeepSeek / OpenAI / OpenRouter / Novita / Fireworks / NvidiaNim / SGLang
```

## 技术栈

| 层级 | 技术 | 版本 | 说明 |
|------|------|------|------|
| TUI 框架 | Ratatui | v0.29 | Widget、布局、样式系统 |
| 终端控制 | Crossterm | v0.28 | 事件、原始模式、括号粘贴 |
| 异步运行时 | Tokio | 1.49 | 全功能异步 |
| HTTP 客户端 | reqwest | 0.13.1 | HTTP/2、rustls、JSON |
| HTTP 服务端 | axum | 0.8.5 | app-server 传输层 |
| ORM/DB | rusqlite | 0.32.1 | 内嵌 SQLite（bundled） |
| 配置 | toml | 0.9.7 | TOML 格式配置文件 |
| CLI | clap | 4.5.54 | 参数解析（derive 宏） |
| 序列化 | serde + serde_json | 1.0 | JSON 处理 |
| 错误处理 | anyhow + thiserror | 1.0 / 2.0 | 灵活错误 + 自定义错误 |
| 日志 | tracing | 0.1 | 结构化日志 |
| UUID | uuid | 1.11 | v4 UUID 生成 |
| 密钥 | keyring | 3.x | OS keyring（三平台） |
| 伪终端 | portable-pty | 0.8 | Shell 工具用 PTY |
| 正则 | regex | 1.11 | 正则表达式 |

## Crate 依赖层级

```
Layer 5 (顶层)
  └── cli               ← deepseek 二进制 dispatcher

Layer 4 (应用层)
  ├── app-server         ← HTTP/SSE 运行时 API
  └── tui                ← 主 TUI 应用（230+ Rust 文件）

Layer 3 (引擎层)
  └── core               ← 代理循环、会话、容量、一致性

Layer 2 (代理层)
  └── agent              ← 供应商注册表、ModelRegistry

Layer 1 (扩展层)
  ├── tools              ← 工具原始类型
  ├── mcp                ← MCP 客户端
  ├── hooks              ← 生命周期钩子
  └── execpolicy         ← 批准/沙盒策略

Layer 0 (基础层)
  ├── protocol           ← 消息/请求/响应数据结构
  ├── config             ← 配置加载（TOML + 环境变量）
  ├── state              ← SQLite 持久化
  └── tui-core           ← 事件驱动状态机框架
```

## 运行模式

| 模式 | 英文 | 说明 |
|------|------|------|
| 代理模式 | Agent | 完整工具支持，需要批准高风险操作 |
| YOLO 模式 | Yolo | 自动批准所有工具，无需确认 |
| 计划模式 | Plan | 只读模式，专注于任务规划编辑 |

## 核心数据流

```
用户输入（键盘 / 命令）
  │
  ▼
Crossterm 事件循环
  │  ├── KeyEvent → Composer 处理
  │  ├── MouseEvent → 选择/滚动
  │  └── PasteEvent → PasteBurst 处理
  │
  ▼
App::update() [app.rs]
  │  ├── 斜杠命令 → command_dispatch()
  │  └── 普通消息 → 加入消息队列
  │
  ▼
Core Engine [core/engine.rs - 74K]
  │  ├── 刷新系统提示词
  │  ├── 自动压缩检查（80% 上下文窗口）
  │  ├── 构建工具目录
  │  └── 容量检查（CapacityController）
  │
  ▼
DeepSeekClient::handle_chat_completion_stream() [client.rs - 84K]
  │  ├── 速率限制检查（TokenBucket）
  │  ├── HTTP POST /v1/chat/completions
  │  └── SSE 字节流
  │
  ▼
parse_sse_chunk()
  │  ├── Text Delta → UI 渲染
  │  ├── Thinking Delta → 思考块渲染
  │  └── ToolUse → 工具路由
  │
  ▼
Tool Routing [tool_routing.rs]
  │  ├── 批准检查 (ApprovalPolicy)
  │  ├── 并发执行（最多 4 个工具）
  │  └── 结果流式返回 UI
  │
  ▼
Quota 更新 + 会话持久化 [state crate]
```

## 支持的 LLM 提供商

| 提供商 | Endpoint | 代表模型 |
|--------|----------|---------|
| DeepSeek（默认） | api.deepseek.com | deepseek-v4-pro, deepseek-v4-flash |
| NVIDIA NIM | integrate.api.nvidia.com/v1 | deepseek-ai/deepseek-v4-pro |
| OpenAI | api.openai.com/v1 | gpt-4.1, gpt-4.1-mini |
| OpenRouter | openrouter.ai/api/v1 | deepseek/deepseek-v4-pro |
| Novita | api.novita.ai/v1 | deepseek/deepseek-v4-pro |
| Fireworks | api.fireworks.ai/inference/v1 | accounts/fireworks/models/deepseek-v4-pro |
| SGLang（自托管） | localhost:30000/v1 | deepseek-ai/DeepSeek-V4-Pro |

## 文档索引

- [[02-crate-analysis\|Crate 职责分析]]
- [[03-tui-components\|TUI 组件系统]]
- [[04-api-client\|API 客户端与流式处理]]
- [[05-tool-system\|工具系统与 MCP]]
- [[06-session-storage\|会话管理与持久化]]
- [[07-insights\|设计洞察与 Legion 参考]]

<!-- @end-section -->
