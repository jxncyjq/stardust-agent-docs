---
id: "analysis-clawcode-overview-001"
title: "Claw Code 项目架构总览"
aliases: ["claw-code overview", "claw 架构总览"]
type: "analysis"
category: "design/analysis/claude"
tags: ["claw-code", "architecture", "rust", "overview"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: null
children: ["analysis-clawcode-rust-002", "analysis-clawcode-python-003", "analysis-clawcode-checklist-004"]
related_docs:
  - id: "analysis-clawcode-rust-002"
    relation: "related_to"
    path: "./02-rust-crates-analysis.md"
  - id: "analysis-clawcode-python-003"
    relation: "related_to"
    path: "./03-python-subsystems-analysis.md"
  - id: "analysis-clawcode-checklist-004"
    relation: "related_to"
    path: "./04-requirements-checklist.md"
---

<!-- @section: overview -->
# Claw Code 项目架构总览

## 项目概述

Claw Code 是一个开源的 CLI AI 代理工具，是 Anthropic Claude Code 的 Rust 重写实现。项目通过命令行与 AI 模型交互，支持代码分析、文件操作、任务编排等多种功能。

- 仓库地址：`ultraworkers/claw-code`
- 主要语言：Rust（主力实现）/ Python（参考移植分析工作空间）
- 二进制名称：`claw`
- 协议支持：Anthropic Messages API / OpenAI Chat Completions 兼容协议

## 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                  rusty-claude-cli (CLI 入口)               │
│         main.rs / input.rs / render.rs / init.rs          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │ commands │  │  tools   │  │ plugins  │  │telemetry│ │
│  │(斜杠命令) │  │(工具框架) │  │(插件系统) │  │ (遥测)  │ │
│  └────┬─────┘  └────┬─────┘  └──────────┘  └─────────┘ │
│       │             │                                    │
│  ┌────┴─────────────┴──────────────────────────────────┐ │
│  │                   runtime (核心运行时)                 │ │
│  │  43 个模块：session / config / permissions / mcp /    │ │
│  │  lsp / bash / sandbox / conversation / oauth / ...   │ │
│  └──────────────────────┬───────────────────────────────┘ │
│                         │                                  │
│  ┌──────────────────────┴───────────────────────────────┐ │
│  │                   api (API 客户端)                     │ │
│  │  Anthropic / OpenAI / xAI / DashScope 多提供商支持     │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                         │
│  ┌────────────────────┐  ┌─────────────────────────────┐ │
│  │  mock-anthropic-   │  │     compat-harness           │ │
│  │  service (测试Mock) │  │     (兼容性校验)               │ │
│  └────────────────────┘  └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

## 两套代码体系

### Rust 实现（主力，`rust/crates/`）

9 个 crate，`rust/crates/**/src/*.rs` 共约 78,700 行 Rust 代码（不含 `tests/`、`benches/`），是 claw CLI 的完整运行实现。

| Crate | 角色 | src 文件数 |
|-------|------|-----------|
| `rusty-claude-cli` | CLI 入口程序 | 4 |
| `runtime` | 核心运行时引擎 | 43 |
| `api` | 多提供商 API 客户端 | 10 |
| `tools` | 工具注册与执行框架 | 3 |
| `commands` | 斜杠命令系统 | 1 |
| `plugins` | 插件生命周期管理 | 3 |
| `telemetry` | 遥测与追踪 | 1 |
| `compat-harness` | 上游兼容性校验 | 1 |
| `mock-anthropic-service` | 测试 Mock 服务 | 2 |

> 数据基于 `rust/crates/<name>/src/*.rs` 的实际文件统计（含 `lib.rs`），不等于 Rust 模块声明数。

### Python 实现（参考分析，`src/`）

顶层 36 个 `.py` 操作型模块 + `reference_data/subsystems/` 下 29 个子系统占位包，用于对照原始 TypeScript 代码库进行移植分析和一致性审计。

## 核心数据流

```
用户输入
  │
  ▼
CLI 解析 (rusty-claude-cli/main.rs)
  │
  ├── 诊断模式 (doctor / status / sandbox / version)
  ├── 管理操作 (init / config / mcp / plugins / skills)
  ├── REPL 交互模式
  └── 一次性 prompt 模式
        │
        ▼
运行时初始化 (config.rs / prompt.rs / bootstrap.rs)
  │
  ▼
对话循环 (conversation.rs)
  │
  ├── 系统提示组装 (prompt.rs + git_context.rs)
  ├── API 调用 (api crate → Anthropic / OpenAI / xAI)
  ├── 流式响应解析 (sse.rs)
  ├── 工具调用分发 (tools crate → 50 个工具规范)
  │     ├── 文件操作 (file_ops.rs)
  │     ├── Bash 执行 (bash.rs + bash_validation.rs)
  │     ├── 任务管理 (task_registry.rs)
  │     ├── MCP 工具 (mcp_tool_bridge.rs)
  │     └── LSP 操作 (lsp_client.rs)
  ├── 权限检查 (permission_enforcer.rs)
  └── 会话压缩 (compact.rs / summary_compression.rs)
        │
        ▼
会话持久化 (session.rs / session_control.rs)
```

## 支持的 AI 提供商

| 提供商 | 协议 | 默认端点 |
|--------|------|----------|
| Anthropic | Anthropic Messages API | api.anthropic.com |
| OpenAI | OpenAI Chat Completions | api.openai.com/v1 |
| xAI (Grok) | OpenAI 兼容 | api.x.ai/v1 |
| DashScope (阿里云) | OpenAI 兼容 | dashscope.aliyuncs.com |
| OpenRouter | OpenAI 兼容 | openrouter.ai/api/v1 |
| Ollama (本地) | OpenAI 兼容 | 127.0.0.1:11434/v1 |

> `api/providers/mod.rs::MODEL_REGISTRY` 还区分推理类模型，对 `o1`/`grok` 等会自动清理 `temperature`/`top_p` 等不兼容采样参数（见 [[04-requirements-checklist|PRV-014]]）。

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[02-rust-crates-analysis|Rust Crate 功能模块分析]]
- [[03-python-subsystems-analysis|Python 子系统功能分析]]
- [[04-requirements-checklist|系统化需求清单]]
<!-- @end-section -->
