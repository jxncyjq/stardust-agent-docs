---
id: "analysis-mission-control-orchestration-003"
title: "Mission Control 编排引擎与调度器"
aliases: ["MC orchestration", "MC编排"]
type: "analysis"
category: "design/analysis/mission-control"
tags: ["mission-control", "orchestration", "scheduler", "aegis", "gateway", "cron", "pipeline"]
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
---

# Mission Control 编排引擎与调度器

<!-- @section: orchestration-patterns -->

## 1. 七种编排模式

### 模式 1：手动指派（Manual Assignment）

**适用场景**：精确控制任务执行者，或测试特定 Agent

```
操作员
  │ 创建任务 + 指定 assigned_to
  ▼
tasks.status = 'assigned'
  │
  ▼ Agent 轮询 GET /api/tasks/queue
tasks.status = 'in_progress'
```

**特点**：最简单，无自动路由。Agent 需要主动轮询队列，或通过心跳收到通知。

---

### 模式 2：队列调度（Queue-Based Dispatch）

**适用场景**：多 Agent 竞争消费任务池，无需指定具体执行者

```
任务进入 tasks.status = 'inbox'
  │
  ├─ Agent A 轮询 → 原子 CLAIM → in_progress
  ├─ Agent B 轮询 → 无可用任务 → 等待
  └─ ...
```

**核心机制**：SQLite 原子 UPDATE-RETURNING 保证无竞态条件（见 [[02-task-agent-lifecycle|§3.3]]）。

**容量控制**：`max_capacity` 参数（默认 1，最大 20），允许 Agent 并行处理多任务。

---

### 模式 3：自动派遣（Auto-Dispatch）

**适用场景**：生产环境，需要 OpenClaw Gateway 支持

```
scheduler.task_dispatch（60s tick）
  │
  ├─ autoRouteInboxTasks()
  │    └─ 匹配 Agent 能力/角色 → assigned
  │
  └─ dispatchAssignedTasks()
       └─ 构建 prompt（SOUL + 任务 + 模型路由）
       └─ 调用 runOpenClaw(['gateway', 'agentTurn', ...])
```

**模型路由规则**（关键词匹配）：

| 条件 | 路由到 |
|------|-------|
| 优先级 `critical` | Opus（最强模型）|
| 关键词含 `debug`, `architect`, `security audit`, `complex` | Opus |
| 关键词含 `format`, `rename`, `ping`, `summarize`, `translate` | Haiku（轻量）|
| 其余 | Agent 默认配置模型 |

---

### 模式 4：质量审查（Aegis Quality Gate）

**Aegis** 是 Mission Control 内置的 AI 质量审查员（通常由高质量模型担任，如 Claude Opus）。

**完整流程**：

```
Agent 完成任务
  │ PUT /api/tasks/{id} → status = 'quality_review'
  │
  ▼
scheduler.aegis_review（60s tick）
  │
  ├─ 查询 quality_review 状态的任务
  ├─ 构建审查 prompt（任务描述 + Agent 输出 + 验收标准）
  ├─ 调用 Aegis（LLM）
  │
  ├─ 返回 "VERDICT: APPROVED"
  │    └─ tasks.status = 'done'
  │    └─ quality_reviews.status = 'approved'
  │
  └─ 返回 "VERDICT: REJECTED" + 反馈意见
       └─ tasks.status = 'in_progress'（Agent 须根据反馈重做）
       └─ quality_reviews.status = 'rejected'
       └─ 循环计数 +1
          │
          └─ 超过 3 次仍 REJECTED
               └─ tasks.status = 'failed'
```

**保护机制**：任何 API 直接将任务设为 `done` 而不经过 Aegis 审核会返回 403。

---

### 模式 5：定时任务（Recurring / Cron）

**两套独立系统**：

**系统 A：内置 Recurring Task**

通过 `tasks.metadata.recurrence` 配置：
```json
{
  "recurrence": {
    "enabled": true,
    "frequency": "weekly",
    "cron": "0 9 * * 1",
    "nextRun": 1746518400
  }
}
```
调度器 `recurring_task_spawn` 在到期时克隆模板任务（保留标题/描述/指派/优先级/metadata，重置状态为 inbox）。

**系统 B：OpenClaw Cron（外部）**

由 OpenClaw 守护进程管理，配置存于 `~/.openclaw/cron/jobs.json`。MC 提供 `/api/cron` 端点用于列表查询和管理，实际执行由 `openclaw cron trigger` CLI 命令完成。

---

### 模式 6：多 Agent 接力（Handoff Pipeline）

**适用场景**：任务需要多个专业 Agent 接力完成（如代码生成→安全审查→部署）

```
Agent A 完成阶段 1
  │ 创建新任务，assigned_to = 'Agent B'
  ▼
Agent B 执行阶段 2
  │ 创建新任务，assigned_to = 'Agent C'
  ▼
...
```

**Pipeline 系统**（更结构化的接力）：

`workflow_pipelines` 表定义步骤序列，`pipeline_runs` 追踪运行状态：

```json
{
  "steps": [
    {"template_id": 1, "on_failure": "stop"},
    {"template_id": 2, "on_failure": "continue"},
    {"template_id": 3, "on_failure": "stop"}
  ]
}
```

每步通过 `runOpenClaw(['agent', '--message', '...', '--timeout', '...'])` 实际执行，可 `start → advance → cancel`。

---

### 模式 7：僵尸任务恢复（Stale Task Requeue）

**触发条件**：`in_progress` 状态超过 10 分钟且对应 Agent 离线

**恢复流程**：
```
scheduler.stale_task_requeue（60s tick）
  │
  ├─ 查找 status='in_progress' AND Agent.status='offline'
  │  AND tasks.updated_at < (now - 600)
  │
  ├─ 次数 < 5 → tasks.status = 'assigned'（重新排队）
  │             dispatch_attempts + 1
  │
  └─ 次数 ≥ 5 → tasks.status = 'failed'
                写 error_message = '自动恢复失败次数已达上限'
```

**防止无限循环**：`dispatch_attempts` 计数器 + 最大尝试次数 5 次硬限制。

<!-- @end-section -->

<!-- @section: scheduler -->

## 2. 调度器详解

### 2.1 调度器架构

调度器在 Next.js 进程启动时通过 `initScheduler()` 初始化，使用 Node.js `setInterval` 每 60 秒执行一次 `tick()`。

**防重入机制**：每个任务有 `running: boolean` 标志，tick 时跳过正在运行的任务。

**动态启用**：各任务是否启用通过 `settings` 表的 `general.*` / `webhooks.*` 键动态读取，每次 tick 都重新查询。

**手动触发**：`triggerTask(id)` 可绕过 nextRun 检查立即执行指定任务。

### 2.2 全部 13 个调度任务

| 任务 ID | 名称 | 间隔 | 首次延迟 | 默认启用 |
|---------|------|------|---------|---------|
| `agent_heartbeat` | Agent 离线检测 | 5min | 5min | ✅ |
| `task_dispatch` | 任务自动路由与派遣 | 60s | 10s | ✅ |
| `aegis_review` | Aegis 质量审核 | 60s | 30s | ✅ |
| `recurring_task_spawn` | 定时任务克隆 | 60s | 20s | ✅ |
| `stale_task_requeue` | 僵尸任务恢复 | 60s | 25s | ✅ |
| `webhook_retry` | Webhook 重试 | 60s | 60s | ✅ |
| `claude_session_scan` | Claude Code 会话扫描 | 60s | 5s | ✅ |
| `skill_sync` | 技能磁盘↔DB 同步 | 60s | 10s | ✅ |
| `local_agent_sync` | 本地 Agent 发现 | 60s | 15s | ✅ |
| `gateway_agent_sync` | 网关 Agent 同步 | 60s | 20s | ✅ |
| `auto_backup` | 数据库自动备份 | 24h | 次日凌晨 3 点 UTC | ❌ |
| `auto_cleanup` | 过期数据清理 | 24h | 次日凌晨 4 点 UTC | ❌ |

### 2.3 task_dispatch 内部逻辑

```
autoRouteInboxTasks()
  └─ 对每个 inbox 任务：
     1. 查找 capability 匹配且 status=idle 的 Agent
     2. 若无匹配，查找 role 匹配的 Agent
     3. 若仍无，保持 inbox（等待下次 tick）
     4. 匹配成功 → UPDATE tasks SET status='assigned', assigned_to=?

dispatchAssignedTasks()
  └─ 对每个 assigned 任务：
     1. 读取 Agent.soul_content（SOUL）
     2. 根据任务关键词确定模型路由
     3. 调用 runOpenClaw(['gateway', 'agentTurn',
          '--agent', agentName,
          '--message', taskPrompt,
          '--model', selectedModel,
          '--session-label', taskId])
     4. 写 spawn_history 记录
```

<!-- @end-section -->

<!-- @section: gateway -->

## 3. 网关系统

### 3.1 网关本质

Gateway 是 Mission Control 与 AI Agent 运行时之间的 **WebSocket 通信桥梁**，负责向 Agent 进程发送任务指令（agentTurn）并接收执行状态。没有网关，MC 只能被动等待 Agent 轮询，无法主动派遣任务。

### 3.2 支持的网关

**OpenClaw Gateway**（主推）：
- 默认端口：`18789`
- 配置文件：`openclaw.json`（含 auth token/password、allowedOrigins）
- MC 自动注册自身为 Dashboard（`registerMcAsDashboard()`，写入 allowedOrigins）
- 健康检查：`GET http://{host}:{port}/health`，响应头 `x-openclaw-version` 用于版本检测
- 版本注意：OpenClaw 2026.3.2+ 默认 tools.profile 改为 `messaging`，MC 需强制 `coding` profile

**Hermes Gateway**：
- 安装目录：`~/.hermes/`
- Docker：`hermes gateway run`（前台 + PID 文件）
- 裸机：systemd 服务
- PID 文件：`~/.hermes/gateway.pid`，日志：`~/.hermes/gateway.log`

### 3.3 连接流程

```
前端 → POST /api/gateways/connect（传 gateway ID）
        │
        ▼
后端解析配置，构造 WebSocket URL
  特殊处理：localhost gateway + 远端浏览器
    → 检测是否通过 Tailscale Serve 代理
    → wss://host/gw 路径模式 或 端口代理
        │
        ▼
读取 token（优先级：ENV > openclaw.json > DB）
        │
        ▼
返回 {ws_url, token, token_set}
        │
        ▼
前端建立 WebSocket 连接
```

### 3.4 健康探测

**端点**：`POST /api/gateways/health`

- 服务端发起（可访问 localhost，浏览器不行）
- 5 秒超时
- SSRF 防护：阻止 `metadata.google.internal` 等云元数据地址
- 结果写入 `gateway_health_logs` 表，更新 `status/latency/last_seen`

### 3.5 数据库结构

```sql
CREATE TABLE gateways (
  id           INTEGER PRIMARY KEY,
  name         TEXT,
  host         TEXT,
  port         INTEGER,
  token        TEXT,
  is_primary   INTEGER DEFAULT 0,  -- 唯一主网关
  status       TEXT,
  last_seen    INTEGER,
  latency      INTEGER,
  sessions_count INTEGER,
  agents_count INTEGER
)
```

<!-- @end-section -->

<!-- @section: framework-adapters -->

## 4. 框架适配器

### 4.1 统一接口

```typescript
interface FrameworkAdapter {
  register(agent: AgentRegistration): Promise<void>
  heartbeat(payload: HeartbeatPayload): Promise<void>
  reportTask(report: TaskReport): Promise<void>
  getAssignments(agentId: string): Promise<Assignment[]>
  disconnect(agentId: string): Promise<void>
}
```

### 4.2 六种框架适配器

| 框架 | 文件 | 特点 |
|------|------|------|
| `openclaw` | `adapters/openclaw.ts` | 原生，直接广播 eventBus |
| `crewai` | `adapters/crewai.ts` | CrewAI 框架接入 |
| `langgraph` | `adapters/langgraph.ts` | LangGraph 框架接入 |
| `autogen` | `adapters/autogen.ts` | AutoGen 框架接入 |
| `claude-sdk` | `adapters/claude-sdk.ts` | Anthropic Claude SDK 原生 |
| `generic` | `adapters/generic.ts` | 通用兜底适配器 |

**设计模式**：所有适配器都将操作翻译为 `eventBus.broadcast()` 调用：
- `register` → `agent.created`
- `heartbeat` / `disconnect` → `agent.status_changed`
- `reportTask` → `task.updated`

### 4.3 任务分配共享逻辑

所有适配器的 `getAssignments()` 调用共享的 `queryPendingAssignments()`：

```sql
SELECT * FROM tasks
WHERE assigned_to = ?
  AND status IN ('assigned', 'in_progress')
ORDER BY
  CASE priority WHEN 'critical' THEN 0 WHEN 'high' THEN 1
                WHEN 'medium' THEN 2 ELSE 3 END,
  created_at ASC
LIMIT 5
```

<!-- @end-section -->

<!-- @section: webhook -->

## 5. Webhook 系统

### 5.1 事件映射

MC 内部事件通过 eventBus 触发出站 Webhook：

| 内部事件 | Webhook 事件类型 |
|---------|----------------|
| `agent.status_changed` | `agent.status_change` |
| `task.created` | `activity.task_created` |
| `task.updated` | `activity.task_updated` |
| `activity.created` | `activity.{data.type}` |
| `notification.created` | `notification.{data.type}` |

订阅 `["*"]` 的 Webhook 接收所有事件。

### 5.2 传输格式

```json
{
  "event": "activity.task_updated",
  "timestamp": 1746700800,
  "data": { ... }
}
```

Headers：
- `X-MC-Event`：事件类型
- `X-MC-Signature: sha256={hmac}`（可选，`timingSafeEqual` 防时序攻击）
- `User-Agent: MissionControl-Webhook/1.0`

### 5.3 重试与熔断机制

**退避表**（±20% 随机抖动）：30s → 5min → 30min → 2h → 8h（最多 5 次）

**熔断器**：连续失败达到阈值 → 自动 `enabled=0`，手动 `reset_circuit: true` 重置。

**SSRF 防护**：禁止目标 URL 为 localhost、私有 IP（10.x/172.16-31.x/192.168.x）、云元数据端点。

**记录保留**：每个 Webhook 保留最近 200 条投递记录（自动剪裁）。

<!-- @end-section -->

<!-- @section: related -->

## 相关文档

- [[index|分析索引]]
- [[02-task-agent-lifecycle|02 任务与 Agent 生命周期]]
- [[04-memory-skills-hub|04 记忆系统与技能 Hub]]
- [[05-security-eval-framework|05 安全框架与 Agent 评估]]
- [[07-insights|07 设计洞察与 Legion 参考]]

<!-- @end-section -->
