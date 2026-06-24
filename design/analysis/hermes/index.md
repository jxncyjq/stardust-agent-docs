---
id: "analysis-hermes-index"
title: "Hermes Agent 分析索引"
aliases: ["hermes index", "Hermes分析索引"]
type: "analysis"
category: "design/analysis/hermes"
tags: ["hermes-agent", "index", "analysis"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-04"
author: "jxncyjq"
status: "review"
parent: null
children:
  - "analysis-hermes-overview-001"
  - "analysis-hermes-runtime-002"
  - "analysis-hermes-tools-003"
  - "analysis-hermes-gateway-004"
  - "analysis-hermes-datamodels-005"
  - "analysis-hermes-insights-006"
  - "analysis-hermes-vs-evolver-007"
---

<!-- @section: index -->
# Hermes Agent 分析文档索引

## 概述

本目录包含对 **Hermes Agent**（Nous Research 开发的自我完善的 AI Agent 框架）的完整架构分析文档。Hermes Agent 是当前功能最丰富的开源 AI Agent 平台，是 Legion AI Agent 运行时引擎的重要参考项目。

- 分析对象：`hermes-agent/`（Python 3.11+, React 19, Ink v6）
- 文档数量：7 份分析文档 + 1 份索引
- 分析深度：系统级（架构、运行时、工具系统、网关、数据层）

## 文档列表

| 序号 | 文档 | 内容 | 状态 |
|------|------|------|------|
| 01 | [[01-overview|项目架构总览]] | 系统定位、技术栈、入口点对比、核心数据流 | review |
| 02 | [[02-agent-runtime|Agent 运行时引擎分析]] | AIAgent 主循环、传输层、提供商适配、上下文压缩 | review |
| 03 | [[03-tools-skills-plugins|工具、技能与插件系统分析]] | ToolRegistry、技能市场、插件钩子、MCP 集成 | review |
| 04 | [[04-gateway-cli-deployment|网关、CLI 与部署分析]] | 19 平台网关、CLI/TUI/Web、Docker 部署 | review |
| 05 | [[05-data-models|状态持久化与数据模型分析]] | SessionDB (SQLite+FTS5)、配置系统、日志 | review |
| 06 | [[06-hermes-insights|Hermes 洞察与 Legion 参考]] | 设计模式提炼、差异化方向、设计建议 | review |
| 07 | [[07-hermes-vs-evolver|Hermes vs Evolver 深度对比]] | 两项目深度对比、Hermes 独特优势、对 Legion 的启示 | review |

## 文档依赖关系

```
01-overview (总览)
  ├── 02-agent-runtime (运行时)
  │     ├── 03-tools-skills-plugins (能力系统)
  │     └── 05-data-models (数据模型)
  ├── 04-gateway-cli-deployment (入口与部署)
  ├── 06-hermes-insights (Legion 参考) ← 汇总全部
  └── 07-hermes-vs-evolver (Evolver 对比) ← 横向对比
```

## 阅读路径

### 路径 1: Agent 引擎理解
`01-overview → 02-agent-runtime → 03-tools-skills-plugins → 05-data-models`

适合快速理解 Agent 核心运行时和工具系统。

### 路径 2: 完整平台理解
`01-overview → 04-gateway-cli-deployment → 03-tools-skills-plugins`

适合了解 Hermes 如何提供多入口（CLI/TUI/Web/Gateway/ACP）。

### 路径 3: Legion 设计参考
`01-overview → 06-hermes-insights → 02-agent-runtime → 03-tools-skills-plugins`

适合从 Hermes 中提炼可复用的设计模式和需要注意的坑点。

### 路径 4: 横向对比分析
`01-overview → 07-hermes-vs-evolver → [[../evolver/07-evolver-vs-hermes|Evolver 对比]]`

适合理解 Hermes 与 Evolver 的差异定位，为 Legion 设计决策提供参考。

## 关键词索引

| 关键词 | 相关文档 |
|--------|----------|
| AIAgent 主循环 | [[02-agent-runtime|02]] |
| Transport 传输层 | [[02-agent-runtime|02]] |
| NormalizedResponse | [[02-agent-runtime|02]] |
| 上下文压缩 | [[02-agent-runtime|02]] |
| 错误分类学 | [[02-agent-runtime|02]] |
| 凭证池 | [[02-agent-runtime|02]] |
| ToolRegistry 自注册 | [[03-tools-skills-plugins|03]] |
| 工具集组合 (includes) | [[03-tools-skills-plugins|03]] |
| 技能渐进式加载 | [[03-tools-skills-plugins|03]] |
| 技能安全扫描 | [[03-tools-skills-plugins|03]] |
| MCP 集成 | [[03-tools-skills-plugins|03]] |
| 插件钩子系统 | [[03-tools-skills-plugins|03]] |
| 多平台网关 | [[04-gateway-cli-deployment|04]] |
| PlatformAdapter | [[04-gateway-cli-deployment|04]] |
| 斜杠命令注册表 | [[04-gateway-cli-deployment|04]] |
| ACP 编辑器集成 | [[04-gateway-cli-deployment|04]] |
| 定时任务系统 | [[04-gateway-cli-deployment|04]] |
| Docker 部署 | [[04-gateway-cli-deployment|04]] |
| SessionDB (SQLite) | [[05-data-models|05]] |
| FTS5 全文搜索 | [[05-data-models|05]] |
| 日志脱敏 | [[05-data-models|05]] |
| 轨迹数据 | [[05-data-models|05]] |
| Evolver 对比 | [[07-hermes-vs-evolver|07]] |

## 技术栈速览

| 层级 | 技术 |
|------|------|
| 语言 | Python 3.11+ |
| Agent 框架 | 自研 (AIAgent) |
| LLM 客户端 | openai, anthropic SDK |
| CLI | prompt_toolkit, Rich, argparse |
| Web 前端 | React 19, TypeScript, Vite, Tailwind CSS v4 |
| TUI 前端 | React 19, Ink v6, TypeScript |
| Web 后端 | FastAPI + WebSocket |
| 数据库 | SQLite (WAL + FTS5) |
| 容器化 | Docker + docker-compose |
| 包管理 | uv (Python), npm (Node.js) |
| 网站 | Docusaurus 3.9 |

## 四项目对比总览

| 维度 | claw-code | new-api | hermes-agent | evolver |
|------|-----------|---------|-------------|---------|
| 语言 | Rust | Go | Python | JavaScript |
| 定位 | Claude Code 替代 | LLM API 网关 | AI Agent 平台 | Agent 自我进化 |
| 参考价值 | Agent 运行时基础 | MaaS 模型调度 | Agent 平台生态 | 进化与知识管理 |
| 核心特色 | MCP/权限/压缩 | 40+渠道适配 | 149 技能市场 + Transport 抽象 | 信号驱动进化 |
| 分析文档 | claude/ (13 份) | maas/ (8 份) | hermes/ (8 份) | evolver/ (8 份) |
| 代码可读性 | 完全开源 | 完全开源 | 完全开源 | 核心混淆 |

## 统计

- 分析覆盖 Python 文件：500+（仅 `agent/` 56 + `tools/` 86 + `gateway/` 56 + `hermes_cli/` 66 + `plugins/` 48 已超 300，含 `tests/` / `environments/` 等更多）
- 核心包：10+
- 内置工具规范：约 68（按 `registry.register()` 顶层调用统计，不含 MCP 动态发现 + 插件注入）
- 内置技能：89（`skills/`，149 计入 `optional-skills/`）
- 支持平台：19（`gateway/platforms/` 核心适配器）
- 分析文档总字数：约 22000 字
- 对比文档：1 份 (Hermes vs Evolver)

<!-- @end-section -->

<!-- @section: related -->
## 相关目录

- [[../claude/index|Claw Code 分析索引]] — Agent 运行时参考项目（Rust）
- [[../maas/index|MaaS 模型调度层分析索引]] — 模型网关参考项目（Go）

<!-- @end-section -->
