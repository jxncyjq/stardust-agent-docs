---
id: "analysis-mission-control-overview-001"
title: "Mission Control 项目总览"
aliases: ["MC overview", "MC总览"]
type: "analysis"
category: "design/analysis/mission-control"
tags: ["mission-control", "nextjs", "sqlite", "architecture", "overview"]
version: "1.0.0"
created: "2026-05-08"
updated: "2026-05-08"
author: "jxncyjq"
status: "review"
parent: "analysis-mission-control-index"
related_docs:
  - id: "analysis-mission-control-index"
    relation: "parent"
    path: "./index.md"
  - id: "analysis-mission-control-task-agent-002"
    relation: "related_to"
    path: "./02-task-agent-lifecycle.md"
  - id: "analysis-mission-control-orchestration-003"
    relation: "related_to"
    path: "./03-orchestration-scheduler.md"
---

# Mission Control 项目总览

<!-- @section: positioning -->

## 1. 系统定位

Mission Control（MC）是 [Builderz Labs](https://github.com/builderz-labs/mission-control) 开源的 **AI Agent 编排仪表盘**，v2.0.1，MIT 许可证。

**核心功能**：管理 AI Agent 舰队——任务分发、状态监控、成本追踪、质量门控、安全审计、记忆知识库。

**设计哲学**：
- **仪表盘优先**：32 个专用 UI 面板，每个业务域独立可视化
- **多路接入**：REST API / CLI / MCP Server（35 工具）/ WebSocket SSE 四套接口
- **被动编排**：MC 不运行 AI 代码，而是通过网关向独立 Agent 进程发送指令
- **可观测性**：全链路事件流 + token 计费 + 信任评分 + 审计日志

<!-- @end-section -->

<!-- @section: tech-stack -->

## 2. 技术栈

| 分类 | 技术 | 版本 | 用途 |
|------|------|------|------|
| 框架 | Next.js | 16 | App Router + API Routes |
| UI 库 | React | 19 | 服务端/客户端组件 |
| 语言 | TypeScript | 5.7 | 全栈类型安全 |
| 数据库 | SQLite（better-sqlite3） | WAL 模式 | 单文件数据库，45 migrations |
| 状态管理 | Zustand | 5 | 客户端全局状态 |
| 样式 | Tailwind CSS | 3 | 原子化 CSS |
| 测试 | Vitest + Playwright | — | 单元（282）+ E2E（295）|
| 容器 | Docker Compose | — | 零配置一键部署 |
| 包管理 | pnpm | — | Monorepo 支持 |

**运行时要求**：Node.js ≥ 22（LTS 推荐），pnpm（corepack enable 自动安装）

**数据目录**：`MISSION_CONTROL_DATA_DIR`（默认 `.data/`），DB 路径：`<dataDir>/mission-control.db`

<!-- @end-section -->

<!-- @section: architecture -->

## 3. 分层架构

```
┌──────────────────────────────────────────────────────────────────┐
│                     接入层（Entry Layer）                          │
│  Browser UI (32 panels) │ REST API │ MCP Server │ SSE EventStream │
└──────────────────────────────────────────────────────────────────┘
                               │
┌──────────────────────────────────────────────────────────────────┐
│                 应用层（App Layer）— Next.js App Router             │
│  src/app/                                                         │
│  ├── page.tsx（主仪表盘）                                           │
│  ├── api/                                                         │
│  │   ├── agents/     tasks/      memory/     skills/              │
│  │   ├── security/   evals/      token-usage/ webhooks/           │
│  │   ├── gateways/   pipelines/  cron/       admin/               │
│  │   └── events（SSE）  auth/    mcp-proxy/                       │
│  └── (auth)/  setup/                                              │
└──────────────────────────────────────────────────────────────────┘
                               │
┌──────────────────────────────────────────────────────────────────┐
│                  核心库层（Core Library）— src/lib/                 │
│  scheduler.ts      — 13 个后台任务，60s tick                        │
│  security-events.ts — 安全事件 + Trust Score                       │
│  agent-evals.ts    — 四层评估框架                                   │
│  skill-sync.ts     — 磁盘↔DB 双向同步                              │
│  skill-registry.ts — 安全扫描 + 注册表客户端                         │
│  memory-utils.ts   — WikiLink / Schema / 知识处理                   │
│  memory-search.ts  — FTS5 全文检索                                  │
│  task-dispatch.ts  — 自动路由 + 分派逻辑                             │
│  event-bus.ts      — 内存 SSE 广播总线                              │
│  adapters/         — 6 种框架适配器                                  │
│  db.ts             — better-sqlite3 连接 + migration runner          │
└──────────────────────────────────────────────────────────────────┘
                               │
┌──────────────────────────────────────────────────────────────────┐
│              数据层（Data Layer）— SQLite WAL                        │
│  45 migrations → 50+ 张表                                          │
│  tasks / agents / token_usage / security_events / eval_runs       │
│  memory_fts（FTS5）/ skills / runs / mcp_call_log                  │
└──────────────────────────────────────────────────────────────────┘
                               │
┌──────────────────────────────────────────────────────────────────┐
│              外部集成层（External Integration）                      │
│  OpenClaw Gateway（WebSocket，端口 18789）                           │
│  Hermes Gateway                                                   │
│  GitHub API（issue/PR 双向同步）                                    │
│  ClawdHub / skills.sh（技能注册表）                                  │
└──────────────────────────────────────────────────────────────────┘
```

<!-- @end-section -->

<!-- @section: data-flow -->

## 4. 核心数据流

### 4.1 任务调度数据流（Auto-Dispatch 模式）

```
操作员创建任务（UI/API）
      │
      ▼
tasks.status = 'inbox'
      │
      ▼ scheduler.task_dispatch（60s）
autoRouteInboxTasks()
  └─ 匹配 Agent 能力/角色 → tasks.status = 'assigned'
      │
      ▼ scheduler.task_dispatch（下次 60s）
dispatchAssignedTasks()
  └─ 调用 OpenClaw Gateway 发送 agentTurn 消息
      │
      ▼ Agent 接收消息，执行任务
      │
      ▼ Agent PUT /api/tasks/{id} → status = 'review'
      │
      ▼ scheduler.aegis_review（60s）
Aegis 审阅 → VERDICT: APPROVED/REJECTED
  ├── APPROVED → status = 'done'
  └── REJECTED → status = 'in_progress'（最多 3 轮）
```

### 4.2 Agent 心跳数据流

```
Agent（每 30s）
      │
      ▼ GET /api/agents/{id}/heartbeat
MC 更新 last_seen，查询工作项：
  - assigned_tasks（已指派的任务）
  - mentions（@提及消息）
  - notifications（通知）
  - urgent_activities（4 小时内紧急活动）
      │
      ▼ 返回 WORK_ITEMS_FOUND / HEARTBEAT_OK
```

### 4.3 实时事件流

```
任意后端操作
      │
      ▼ eventBus.broadcast(eventType, payload)
SSE 内存总线（event-bus.ts）
      │
      ├─→ 浏览器 SSE（GET /api/events）→ UI 实时更新
      ├─→ Webhook 触发器 → 出站 HTTP 推送
      └─→ 安全事件记录 → 信任分重算
```

<!-- @end-section -->

<!-- @section: deployment -->

## 5. 部署方式

### 5.1 本地开发

```bash
pnpm install
pnpm dev              # localhost:3000，热重载
# 首次访问 /setup 创建管理员账户
```

### 5.2 生产部署（独立模式）

```bash
pnpm build
node .next/standalone/server.js
# 注意：standalone 模式使用 node 命令，不用 pnpm start
```

### 5.3 Docker（推荐）

```bash
docker compose up                           # 零配置
# 强化模式（安全加固）：
docker compose -f docker-compose.yml -f docker-compose.hardened.yml up -d
```

### 5.4 环境变量

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `AUTH_SECRET` | JWT 签名密钥 | 首次运行自动生成 |
| `API_KEY` | 全局 API Key | 首次运行自动生成 |
| `AUTH_USER` / `AUTH_PASS` | 无头/CI 模式预置管理员 | — |
| `MISSION_CONTROL_DATA_DIR` | 数据目录 | `.data/` |
| `NEXT_PUBLIC_GATEWAY_OPTIONAL` | 无网关独立运行 | `false` |
| `MC_PROXY_AUTH_HEADER` | 代理认证 Header 名 | — |
| `MC_PROXY_AUTH_TRUSTED_IPS` | 代理认证可信 IP | — |

<!-- @end-section -->

<!-- @section: key-modules -->

## 6. 关键目录结构

```
mission-control/
├── src/
│   ├── app/
│   │   ├── api/                # 101 个 REST 端点
│   │   │   ├── agents/         # Agent CRUD + 心跳 + 队列 + comms
│   │   │   ├── tasks/          # 任务 CRUD + quality-review
│   │   │   ├── memory/         # 记忆 CRUD + 图谱 + 搜索 + 处理
│   │   │   ├── skills/         # 技能 CRUD + 注册表 + 同步
│   │   │   ├── security/       # 安全事件 + 信任分 + 态势
│   │   │   ├── evals/          # 评估运行 + 优化建议
│   │   │   ├── token-usage/    # Token 计费统计
│   │   │   ├── gateways/       # 网关管理 + 连接 + 健康检查
│   │   │   ├── pipelines/      # Pipeline 编排
│   │   │   ├── cron/           # OpenClaw Cron 集成
│   │   │   ├── webhooks/       # 出站 Webhook
│   │   │   ├── events/         # SSE 事件流
│   │   │   └── auth/           # 认证（local/google/proxy）
│   │   └── (main)/             # 32 个 UI 面板页面
│   ├── components/
│   │   └── panels/             # 40 个面板组件
│   └── lib/                    # 核心引擎库
├── scripts/
│   └── mc-mcp-server.cjs       # 35 工具 MCP Server（零依赖）
├── docs/
│   ├── orchestration.md        # 7 种编排模式
│   └── agent-setup.md          # Agent 接入指南
├── openapi.json                # OpenAPI 1.3.0 规范（14 个标签）
└── .data/                      # SQLite DB + 运行时状态（gitignored）
```

<!-- @end-section -->

<!-- @section: scale -->

## 7. 规模指标

| 指标 | 数值 |
|------|------|
| 版本 | v2.0.1 |
| 数据库 migrations | 50 个（编号跳过 030/031，实有约 48 个） |
| REST API 端点 | 101 个 |
| MCP Server 工具 | 35 个 |
| UI 面板组件 | 40 个 |
| 单元测试 | 282 个（Vitest）|
| E2E 测试 | 295 个（Playwright）|
| 支持 Agent 框架 | 6 种（OpenClaw/CrewAI/LangGraph/AutoGen/ClaudeSDK/Generic）|
| 本地 Agent 发现目录 | 5 个（~/.agents / ~/.claude / ~/.codex / ~/.hermes / ~/.openclaw）|
| 技能注册表 | 3 个（ClawdHub + skills.sh + Awesome OpenClaw）|
| UI 语言 | 10 种（含中文、日文、阿拉伯文）|
| 任务状态数 | 6 个（inbox→assigned→in_progress→quality_review→done）|
| Agent 状态数 | 5 个（offline/idle/busy/sleeping/error）|
| 后台调度任务 | 13 个（Node.js setInterval）|

<!-- @end-section -->

<!-- @section: related -->

## 相关文档

- [[index|分析索引]]
- [[02-task-agent-lifecycle|02 任务与 Agent 生命周期]]
- [[03-orchestration-scheduler|03 编排引擎与调度器]]
- [[04-memory-skills-hub|04 记忆系统与技能 Hub]]
- [[05-security-eval-framework|05 安全框架与 Agent 评估]]
- [[06-data-models-api|06 数据模型与 API 参考]]
- [[07-insights|07 设计洞察与 Legion 参考]]

<!-- @end-section -->
