---
id: "analysis-mission-control-index"
title: "Mission Control 分析文档索引"
aliases: ["mission-control index", "MC分析索引"]
type: "analysis"
category: "design/analysis/mission-control"
tags: ["mission-control", "nextjs", "agent-dashboard", "orchestration", "sqlite"]
version: "1.0.0"
created: "2026-05-08"
updated: "2026-05-08"
author: "jxncyjq"
status: "review"
parent: null
children:
  - "analysis-mission-control-overview-001"
  - "analysis-mission-control-task-agent-002"
  - "analysis-mission-control-orchestration-003"
  - "analysis-mission-control-memory-skills-004"
  - "analysis-mission-control-security-eval-005"
  - "analysis-mission-control-data-api-006"
  - "analysis-mission-control-insights-007"
---

<!-- @section: overview -->
# Mission Control 分析文档索引

## 分析对象概述

**Mission Control** 是 [Builderz Labs](https://github.com/builderz-labs/mission-control) 开源的 AI Agent 编排仪表盘，v2.0.1，MIT 许可证。

- **定位**：AI Agent 舰队的控制中心——任务分发、状态监控、成本追踪、质量门控、安全审计
- **技术栈**：Next.js 16 + React 19 + TypeScript 5.7 + SQLite (WAL) + Zustand 5
- **规模**：45 个数据库 migration，101 个 REST API 端点，32 个 UI 面板，577 个测试（282 单元 + 295 E2E），10 种 UI 语言
- **接入方式**：REST API / CLI / MCP Server（35 个工具）/ WebSocket SSE 实时推送

## 系统定位图

```
外部 Agent (Claude Code / CrewAI / LangGraph / AutoGen / Codex)
    │
    │  REST API / WebSocket / MCP Server
    ▼
┌─────────────────────────────────────────────────────────────┐
│                   Mission Control Dashboard                   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                   32 个 UI 面板                       │   │
│  │  任务看板 | Agent管理 | 记忆图谱 | 安全审计 |          │   │
│  │  成本追踪 | 技能Hub | Cron调度 | 流水线 | 告警 | ...  │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                   │
│  ┌───────────────────────▼──────────────────────────────┐   │
│  │              核心引擎（src/lib/）                      │   │
│  │  scheduler.ts | security-events.ts | agent-evals.ts  │   │
│  │  skill-sync.ts | skill-registry.ts | adapters/       │   │
│  └───────────────────────┬──────────────────────────────┘   │
│                          │                                   │
│  ┌───────────────────────▼──────────────────────────────┐   │
│  │          SQLite (WAL) — 45 个 migration               │   │
│  │  45 张核心表：tasks / agents / skills / security /    │   │
│  │  memory / projects / tenants / eval_runs / ...       │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
         │                          │
    OpenClaw Gateway          外部集成
    (WebSocket)           GitHub / Webhooks / MCP
```

## 文档列表

| 文档 | 内容 | 关键词 |
|------|------|--------|
| [[01-overview\|01 项目总览]] | 技术栈、分层架构、核心数据流、部署方式 | Next.js、SQLite、Docker |
| [[02-task-agent-lifecycle\|02 任务与 Agent 生命周期]] | 任务状态机、Agent 注册/心跳/SOUL、项目/工单系统 | Kanban、心跳、SOUL.md |
| [[03-orchestration-scheduler\|03 编排引擎与调度器]] | 7 种编排模式、Aegis 质量门控、模型路由、cron、网关 | auto-dispatch、Aegis、Gateway |
| [[04-memory-skills-hub\|04 记忆系统与技能 Hub]] | 记忆图谱、技能安全扫描、注册表、双向磁盘同步 | memory-graph、skills、security-scan |
| [[05-security-eval-framework\|05 安全框架与 Agent 评估]] | 信任评分、4 层评估、MCP 审计、秘密检测 | trust-score、eval、injection |
| [[06-data-models-api\|06 数据模型与 API 参考]] | 全部 45 张表结构、101 个端点、MCP 35 工具 | schema、REST、MCP |
| [[07-insights\|07 设计洞察与 Legion 参考]] | 可直接复用的设计模式、Legion 差异化方向、坑点 | Legion、design-reference |

## 阅读路径

### 路径 A：了解任务编排机制（30 分钟）
[[02-task-agent-lifecycle|02]] → [[03-orchestration-scheduler|03]] → [[07-insights|07]]

### 路径 B：了解安全与评估体系（20 分钟）
[[05-security-eval-framework|05]] → [[04-memory-skills-hub|04]] → [[07-insights|07]]

### 路径 C：Legion 设计参考（完整，60 分钟）
[[01-overview|01]] → [[02-task-agent-lifecycle|02]] → [[03-orchestration-scheduler|03]] → [[05-security-eval-framework|05]] → [[07-insights|07]]

## 关键数字速查

| 指标 | 数值 |
|------|------|
| 版本 | v2.0.1 |
| 数据库 migration 数 | 45 个 |
| REST API 端点数 | 101 个 |
| MCP Server 工具数 | 35 个 |
| UI 面板数 | 32 个 |
| 测试数 | 577（单元 282 + E2E 295）|
| 支持的 Agent 框架 | 5+（OpenClaw/CrewAI/LangGraph/AutoGen/Claude SDK）|
| 本地 Agent 发现目录 | 5 个（~/.agents / ~/.claude / ~/.codex / ~/.hermes / ~/.openclaw）|
| 技能注册表 | 2 个（ClawdHub + skills.sh）|
| UI 语言 | 10 种（含中文、日文、阿拉伯文）|
| 任务状态数 | 6 个（inbox→assigned→in_progress→review→quality_review→done）|
| Agent 状态数 | 5 个（offline/idle/busy/sleeping/error）|

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[../hermes/01-overview|Hermes Agent 分析]]
- [[../deepseek-tui/01-overview|DeepSeek-TUI 分析]]
- [[../evolver/01-overview|Evolver 分析]]
- [[../claude/01-overview|Claw Code 分析]]
- [[../../architecture/Legion|Legion V3.0 项目方案]]
- [[../../architecture/agent-engine-design|Legion Agent 引擎设计方案]]

<!-- @end-section -->
