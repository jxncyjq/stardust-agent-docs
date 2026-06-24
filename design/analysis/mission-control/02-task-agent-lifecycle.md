---
id: "analysis-mission-control-task-agent-002"
title: "Mission Control 任务与 Agent 生命周期"
aliases: ["MC task lifecycle", "MC任务生命周期"]
type: "analysis"
category: "design/analysis/mission-control"
tags: ["mission-control", "task-state-machine", "agent-lifecycle", "soul", "heartbeat", "project"]
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
  - id: "analysis-mission-control-orchestration-003"
    relation: "related_to"
    path: "./03-orchestration-scheduler.md"
---

# Mission Control 任务与 Agent 生命周期

<!-- @section: task-state-machine -->

## 1. 任务状态机

### 1.1 状态转换图

```
                  创建任务
                     │
                     ▼
                  ┌──────┐
                  │inbox │  ← stale_requeue（恢复）
                  └──┬───┘
                     │ autoRouteInboxTasks()
                     │ 或手动 assigned_to 赋值
                     ▼
                ┌──────────┐
                │ assigned │
                └────┬─────┘
                     │ Agent 原子 claim（queue poll）
                     │ 或 dispatchAssignedTasks()
                     ▼
              ┌─────────────┐
              │ in_progress │ ←──── 重新执行
              └──────┬──────┘       │
                     │              │ Aegis REJECTED（≤3次）
              Agent 提交审核          │
                     │              │
                     ▼              │
           ┌──────────────────┐     │
           │ quality_review   │─────┘
           └────────┬─────────┘
                    │ VERDICT: APPROVED
                    │
                    ▼
                 ┌──────┐
                 │ done │
                 └──────┘
```

**废弃状态**：`review` 在 Migration 003 迁移为 `quality_review`，代码中已不再写入该状态。

### 1.2 状态详解

| 状态 | 含义 | 可转入来源 |
|------|------|----------|
| `inbox` | 新建待路由，或从卡死恢复 | 创建时；stale_requeue |
| `assigned` | 已路由到 Agent，等待执行 | inbox（自动路由）；直接创建时指定 assigned_to |
| `in_progress` | Agent 正在执行 | assigned（原子 claim）；quality_review（Aegis 驳回）|
| `quality_review` | Agent 完成，等待 Aegis 审核 | in_progress（Agent 主动提交）|
| `done` | 完成，不可逆终态 | quality_review（Aegis 批准）|

### 1.3 关键约束

**完成保护**：任务从任意状态移入 `done` 必须在 `quality_reviews` 表中存在 `reviewer='aegis'` 且 `status='approved'` 的记录，否则 PUT /api/tasks 返回 **403**。

**首次完成时间**：`completed_at` 通过 `COALESCE(completed_at, ?)` 保证首次设定后不可覆盖。

**派遣计数**：`dispatch_attempts` 字段记录调度尝试次数，用于防止无限重试。

**任务结果字段**（Migration 026）：
- `outcome`：任务结果摘要
- `error_message`：失败原因
- `resolution`：解决方案描述
- `feedback_rating`（1-5）+ `feedback_notes`：人工评分
- `retry_count`：手动重试次数

### 1.4 优先级系统

优先级枚举：`low` / `medium` / `high` / `urgent` / `critical`

队列拉取排序：`critical=0, high=1, medium=2, low=3`（按优先级 + due_date ASC + created_at ASC）

<!-- @end-section -->

<!-- @section: task-schema -->

## 2. 任务数据模型

### 2.1 核心字段

```sql
CREATE TABLE tasks (
  id            INTEGER PRIMARY KEY,
  title         TEXT NOT NULL,
  description   TEXT,
  status        TEXT DEFAULT 'inbox',          -- 状态机
  priority      TEXT DEFAULT 'medium',          -- low/medium/high/urgent/critical
  assigned_to   TEXT,                           -- Agent 名称（字符串，非 FK）
  created_by    TEXT,
  project_id    INTEGER REFERENCES projects(id),
  project_ticket_no INTEGER,                   -- 项目内递增序号（如 #42）

  -- 结果追踪（Migration 026）
  outcome       TEXT,
  error_message TEXT,
  resolution    TEXT,
  feedback_rating   INTEGER,
  feedback_notes    TEXT,
  retry_count   INTEGER DEFAULT 0,
  completed_at  INTEGER,

  -- GitHub 集成（Migration 028）
  github_issue_number INTEGER,
  github_repo   TEXT,
  github_synced_at INTEGER,
  github_branch TEXT,
  github_pr_number INTEGER,
  github_pr_state  TEXT,

  -- 多租户
  workspace_id  INTEGER REFERENCES workspaces(id),

  -- 灵活扩展
  tags          TEXT DEFAULT '[]',             -- JSON 数组
  metadata      TEXT DEFAULT '{}',             -- JSON（含 recurrence 配置）
  
  dispatch_attempts INTEGER DEFAULT 0,
  created_at    INTEGER DEFAULT (unixepoch()),
  updated_at    INTEGER DEFAULT (unixepoch())
)
```

### 2.2 递归任务配置（metadata.recurrence）

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

当调度器检测到 `metadata.recurrence.enabled=1` 时，`recurring_task_spawn` 任务会在下次时间到达时克隆模板任务创建新的子任务。

<!-- @end-section -->

<!-- @section: agent-lifecycle -->

## 3. Agent 生命周期

### 3.1 注册（Register）

**端点**：`POST /api/agents/register`

**幂等设计**：同名 Agent 已存在时，更新 `status='idle'`、`last_seen`，返回 `{registered: false}`；新 Agent 完整创建。

**限流**：5 次/分钟/IP

**注册载荷**：
```json
{
  "name": "codex-agent-01",
  "role": "developer",
  "capabilities": ["code_review", "refactoring"],
  "framework": "claude-sdk"
}
```

**注册后**：写入 `activities` + `audit_log`，广播 SSE `agent.created`。

### 3.2 心跳（Heartbeat）

**端点**：`GET /api/agents/{id}/heartbeat`（轮询）/ `POST`（增强版）

**更新行为**：更新 `agents.last_seen`，设置 `status='idle'`

**工作项发现**（过去 4 小时窗口）：
- `assigned_tasks`：assigned_to = 当前 Agent 的任务列表
- `mentions`：消息中 @提及当前 Agent 的项
- `notifications`：通知列表
- `urgent_activities`：type 含 urgent 的活动

**响应示例**：
```json
{
  "status": "WORK_ITEMS_FOUND",
  "work_items": {
    "assigned_tasks": [...],
    "mentions": [...],
    "notifications": [],
    "urgent_activities": []
  }
}
```

**POST 增强版**额外支持：
- `connection_id`：更新 `direct_connections.last_heartbeat`
- `token_usage`：Inline Token 消耗上报（写入 `token_usage` 表）

### 3.3 任务拉取（Queue Poll）

**端点**：`GET /api/tasks/queue?agent={name}&max_capacity={n}`

**原子 claim 设计**（SQLite WAL 特性保证原子性）：

```sql
UPDATE tasks
SET status = 'in_progress',
    assigned_to = ?,
    updated_at = unixepoch()
WHERE id = (
  SELECT id FROM tasks
  WHERE status IN ('assigned', 'inbox')
    AND (assigned_to IS NULL OR assigned_to = ?)
  ORDER BY
    CASE priority WHEN 'critical' THEN 0 WHEN 'high' THEN 1 
                  WHEN 'medium' THEN 2 ELSE 3 END,
    due_date ASC NULLS LAST,
    created_at ASC
  LIMIT 1
)
RETURNING *
```

**容量控制逻辑**：
1. 若 Agent 有 `in_progress` 任务 → 返回继续执行当前任务（`continue_current`）
2. 若 in_progress 数 ≥ max_capacity → 返回容量已满（`at_capacity`）
3. 否则原子拉取一条任务

### 3.4 离线检测

**触发方式**：调度器 `agent_heartbeat` 任务每 5 分钟执行

**检测条件**：`status != 'offline'` 且 `last_seen < (now - timeout_minutes × 60)`

**后续动作**：
1. 批量 UPDATE agents.status = 'offline'
2. 写 `activities` 记录
3. 写 `notifications`（告知相关方）
4. 写 `audit_log`
5. 触发 `stale_task_requeue`：该 Agent 的 in_progress 任务回退到 assigned

### 3.5 Agent 状态机

```
        注册
          │
          ▼
        idle  ←──── 任务完成 / 心跳后无任务
          │
          │ 拉取任务
          ▼
        busy  ←──── 任务执行中
          │
          │ 明确休眠请求
          ▼
      sleeping
          │
          │ 无心跳超时
          ▼
       offline ←──── 超时检测
          │
          │ 重新注册/心跳
          ▼
        idle

error（任意状态均可转入）
```

<!-- @end-section -->

<!-- @section: soul-system -->

## 4. SOUL 系统

### 4.1 概念

SOUL.md 是 **Agent 的人设与行为配置文档**，以 Markdown 格式编写，定义 Agent 的角色定位、核心价值观、行为准则、技能边界等。Mission Control 在 auto-dispatch 时将 SOUL 内容注入到派遣 prompt 中，让 Agent 的行为符合预期设定。

### 4.2 双层持久化

**读取优先级**：workspace 文件 > 数据库

```
优先级 1：workspace 文件
  agents.workspace_path 目录下的 soul.md 或 SOUL.md

优先级 2：数据库
  agents.soul_content 字段（fallback）
```

### 4.3 写入策略

PUT 请求逻辑：
1. 尝试写入 workspace 文件（`resolveWithin` 防路径穿越）
2. 同步更新数据库 `soul_content` 字段
3. 若 workspace 不可用，仅更新数据库，记录 warn 日志

### 4.4 模板系统

配置路径：`config.soulTemplatesDir`，文件格式：`*.md`

占位符替换：
| 占位符 | 替换内容 |
|--------|---------|
| `{{AGENT_NAME}}` | Agent 名称 |
| `{{AGENT_ROLE}}` | Agent 角色 |
| `{{TIMESTAMP}}` | 当前 ISO 时间 |

**内置模板**：
- `developer`：具备 read/write/edit/exec/bash 权限的开发者人设
- `researcher`：只读 + web 搜索的研究员人设
- `reviewer`：只读代码审查员人设

### 4.5 SOUL 与 dispatch 的关联

Auto-Dispatch 模式下，调度器向 Gateway 发送 agentTurn 消息时：

```
系统提示 = SOUL.md 内容 + 任务描述 + 项目上下文
```

这使不同 Agent 在执行同类任务时产生差异化行为（如安全审计专员 vs 开发工程师的不同处理风格）。

<!-- @end-section -->

<!-- @section: project-system -->

## 5. 项目与工单系统

### 5.1 项目（Project）

**字段**：`id`, `name`, `slug`, `ticket_prefix`, `ticket_counter`（原子递增）

**GitHub 集成**：`github_repo`, `github_sync_enabled`, `github_default_branch`

**团队配置**：`project_agent_assignments` 表（项目-Agent 多对多），记录哪些 Agent 负责该项目。

### 5.2 工单号（Ticket Number）

每个项目维护独立的 `ticket_counter`，任务创建时原子递增并写入 `project_ticket_no`。

显示格式：`{ticket_prefix}-{ticket_no}`，例如 `MC-42`。

存量任务迁移（Migration 024）：已有任务归入系统自动创建的 `general` 项目。

### 5.3 GitHub 双向同步

**同步方向**：
- GitHub Issues → MC Tasks（创建/更新）
- MC Tasks → GitHub Issues（状态变更回写）
- PR 信息：`github_pr_number`, `github_pr_state` 字段追踪关联 PR

**同步历史**：`github_syncs` 表记录每次同步的时间、结果和错误信息。

<!-- @end-section -->

<!-- @section: related -->

## 相关文档

- [[index|分析索引]]
- [[01-overview|01 项目总览]]
- [[03-orchestration-scheduler|03 编排引擎与调度器]]
- [[06-data-models-api|06 数据模型与 API 参考]]

<!-- @end-section -->
