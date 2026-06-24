---
id: "agent-sqlite-schema-001"
title: "Legion Agent agent.db 数据结构"
type: "reference"
category: "backend/agent"
tags: ["agent", "sqlite", "schema", "database"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "agent-index-001"
    relation: "related_to"
    path: "./index.md"
---

# Legion Agent agent.db 数据结构

当配置 `storage.driver=sqlite` 时，Legion Agent 会在 `storage.path` 指定位置创建 `agent.db`。当前 SQLite schema 版本为 `1`，由 `internal/storage/sqlite.go` 中的 `schemaStatements` 初始化。

## 总览

| 表 | 职责 |
|----|------|
| `schema_migrations` | 记录数据库 schema 版本 |
| `tasks` | 保存任务主记录 |
| `task_locks` | 保存任务调度锁和过期时间 |
| `agent_sessions` | 保存 TUI/Agent 会话主记录 |
| `conversation_turns` | 保存会话中的 user/assistant turn |
| `task_runs` | 保存任务执行记录 |
| `audit_events` | 保存审计事件和 hash chain 摘要 |
| `runtime_events` | 保存运行时事件流 |
| `skills` | 保存已安装技能元数据和内容 |
| `skill_scan_findings` | 保存技能安全扫描发现 |
| `capability_assets` | 保存 Gene/Capsule 能力资产 |
| `evolution_events` | 保存 GEP 学习进化事件 |
| `workflow_states` | 保存工作流定义、结果和等待状态 |
| `quality_history` | 保存质量评估、信任快照和退化决策历史 |

## 通用约定

- 时间字段使用 UTC `RFC3339Nano` 字符串存储。
- JSON 数组或复杂对象以 `TEXT` 存储，例如 `tags`、`gene_ids`、`definition_json`、`result_json`。
- 当前 schema 未声明数据库外键，读写层通过业务 ID 维护关联。
- `id INTEGER PRIMARY KEY AUTOINCREMENT` 用于追加型流水表；业务主表多使用文本 ID。

## 表结构说明

### `schema_migrations`

记录已应用的 schema 版本。

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | `INTEGER PRIMARY KEY` | schema 版本号，当前为 `1` |
| `applied_at` | `TEXT NOT NULL` | 迁移应用时间 |

### `tasks`

任务主表，保存 Agent 任务的输入、状态和归属信息。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `TEXT PRIMARY KEY` | 任务 ID |
| `company_id` | `TEXT NOT NULL` | company/tenant 边界标识 |
| `agent_id` | `TEXT NOT NULL` | 执行 Agent ID |
| `status` | `TEXT NOT NULL` | 任务状态，例如 pending/running/done/failed |
| `input` | `TEXT NOT NULL` | 任务输入 |
| `max_iterations` | `INTEGER NOT NULL` | 最大迭代次数 |
| `created_at` | `TEXT NOT NULL` | 创建时间 |

### `task_locks`

任务调度锁表，用于防止多个 worker 同时执行同一任务。

| 字段 | 类型 | 说明 |
|------|------|------|
| `task_id` | `TEXT PRIMARY KEY` | 被锁定的任务 ID |
| `owner_id` | `TEXT NOT NULL` | 锁持有者 |
| `expires_at` | `TEXT NOT NULL` | 锁过期时间，过期后可被抢占 |

### `agent_sessions`

Agent 会话主表。P22 起，TUI 启动会创建或恢复当前 session，并在 footer 中显示 session ID。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `TEXT PRIMARY KEY` | session ID |
| `company_id` | `TEXT NOT NULL` | company/tenant 边界标识 |
| `agent_id` | `TEXT NOT NULL` | 默认 Agent ID |
| `title` | `TEXT NOT NULL` | 会话标题 |
| `created_at` | `TEXT NOT NULL` | 创建时间 |
| `updated_at` | `TEXT NOT NULL` | 最近 turn 写入时间 |

### `conversation_turns`

会话 turn 表。每轮 TUI 对话会写入一条 user turn 和一条 assistant turn；多 Agent 调用会保留实际 `agent_id` 与 `model_profile`。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `TEXT PRIMARY KEY` | turn ID |
| `session_id` | `TEXT NOT NULL` | 所属 session ID |
| `task_id` | `TEXT NOT NULL` | 关联 task/run ID |
| `agent_id` | `TEXT NOT NULL` | 产生该 turn 的 Agent ID |
| `model_profile` | `TEXT NOT NULL` | 使用的 MaaS profile |
| `role` | `TEXT NOT NULL` | `user` 或 `assistant` |
| `content` | `TEXT NOT NULL` | turn 内容，按配置可截断 |
| `created_at` | `TEXT NOT NULL` | 创建时间 |

### `task_runs`

任务执行记录表，一个 task 可以有多次 run。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `TEXT PRIMARY KEY` | run ID |
| `task_id` | `TEXT NOT NULL` | 关联任务 ID |
| `agent_id` | `TEXT NOT NULL` | 执行 Agent ID |
| `started_at` | `TEXT NOT NULL` | 开始时间 |
| `ended_at` | `TEXT NOT NULL` | 结束时间 |
| `result` | `TEXT NOT NULL` | 执行结果文本 |

### `audit_events`

审计事件表，保存关键动作的不可变审计摘要。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `TEXT PRIMARY KEY` | 审计事件 ID |
| `request_id` | `TEXT NOT NULL` | 请求追踪 ID |
| `subject_type` | `TEXT NOT NULL` | 主体类型，例如 task/workflow/skill |
| `subject_id` | `TEXT NOT NULL` | 主体 ID |
| `action` | `TEXT NOT NULL` | 动作名称 |
| `hash` | `TEXT NOT NULL` | 审计 hash 摘要 |
| `created_at` | `TEXT NOT NULL` | 创建时间 |

### `runtime_events`

运行时事件流水表，用于 TUI、SSE 或调试读取。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `INTEGER PRIMARY KEY AUTOINCREMENT` | 自增事件序号 |
| `type` | `TEXT NOT NULL` | 事件类型 |
| `task_id` | `TEXT NOT NULL` | 关联任务 ID |
| `message` | `TEXT NOT NULL` | 事件消息 |
| `created_at` | `TEXT NOT NULL` | 创建时间 |

### `skills`

技能表，保存已安装技能的 manifest 信息和内容。主键为 `(id, version)`。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `TEXT NOT NULL` | 技能 ID |
| `name` | `TEXT NOT NULL` | 技能名称 |
| `source` | `TEXT NOT NULL` | 来源，例如 local/registry |
| `version` | `TEXT NOT NULL` | 技能版本 |
| `path` | `TEXT NOT NULL` | 安装路径 |
| `hash` | `TEXT NOT NULL` | 内容 SHA-256 |
| `risk_level` | `TEXT NOT NULL` | 风险等级 |
| `status` | `TEXT NOT NULL` | 技能状态，例如 active/quarantined |
| `tags` | `TEXT NOT NULL` | JSON 字符串数组 |
| `summary` | `TEXT NOT NULL` | 技能摘要 |
| `content` | `TEXT NOT NULL` | 技能正文 |

### `skill_scan_findings`

技能安全扫描发现表，安装或同步技能时写入。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `INTEGER PRIMARY KEY AUTOINCREMENT` | 自增发现 ID |
| `skill_id` | `TEXT NOT NULL` | 关联技能 ID |
| `rule_id` | `TEXT NOT NULL` | 触发规则 ID |
| `severity` | `TEXT NOT NULL` | 严重级别 |
| `message` | `TEXT NOT NULL` | 扫描消息 |
| `location` | `TEXT NOT NULL` | 发现位置 |

### `capability_assets`

能力资产表，同时承载 Gene 和 Capsule。通过 `asset_type` 区分记录类型，主键为 `(id, asset_type)`。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `TEXT NOT NULL` | 资产 ID |
| `asset_type` | `TEXT NOT NULL` | `gene` 或 `capsule` |
| `version` | `TEXT NOT NULL DEFAULT ''` | Gene 版本 |
| `status` | `TEXT NOT NULL DEFAULT ''` | Gene 状态 |
| `gene_ids` | `TEXT NOT NULL DEFAULT '[]'` | Capsule 关联 Gene ID 数组 |
| `query` | `TEXT NOT NULL DEFAULT ''` | Capsule 查询文本 |
| `tags` | `TEXT NOT NULL DEFAULT '[]'` | 标签 JSON 数组 |
| `match_text` | `TEXT NOT NULL DEFAULT ''` | Gene 匹配条件 |
| `use_when` | `TEXT NOT NULL DEFAULT ''` | Gene 使用条件 |
| `plan` | `TEXT NOT NULL DEFAULT ''` | Gene 行动计划 |
| `avoid` | `TEXT NOT NULL DEFAULT ''` | Gene 避免事项 |
| `constraints_text` | `TEXT NOT NULL DEFAULT ''` | Gene 约束 |
| `validation` | `TEXT NOT NULL DEFAULT ''` | Gene 验证方式 |
| `outcome` | `TEXT NOT NULL DEFAULT ''` | Capsule 结果 |
| `success_rate` | `REAL NOT NULL DEFAULT 0` | 成功率 |
| `success_count` | `INTEGER NOT NULL DEFAULT 0` | 成功次数 |
| `failure_count` | `INTEGER NOT NULL DEFAULT 0` | 失败次数 |
| `confidence` | `REAL NOT NULL DEFAULT 0` | Capsule 置信度 |
| `created_at` | `TEXT NOT NULL DEFAULT ''` | 创建时间 |
| `updated_at` | `TEXT NOT NULL DEFAULT ''` | 更新时间 |

### `evolution_events`

GEP 学习进化事件表，记录进化周期中的阶段、证据和决策。

| 字段 | 类型 | 说明 |
|------|------|------|
| `event_id` | `TEXT PRIMARY KEY` | 事件 ID |
| `cycle_id` | `TEXT NOT NULL` | 进化周期 ID |
| `stage` | `TEXT NOT NULL` | 进化阶段 |
| `agent_id` | `TEXT NOT NULL` | Agent ID |
| `asset_id` | `TEXT NOT NULL` | 关联能力资产 ID |
| `evidence_hash` | `TEXT NOT NULL` | 证据 hash |
| `decision` | `TEXT NOT NULL` | 决策结果 |
| `created_at` | `TEXT NOT NULL` | 创建时间 |

### `workflow_states`

工作流状态表，用于保存可恢复的 workflow 定义和执行结果。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `TEXT PRIMARY KEY` | workflow ID |
| `status` | `TEXT NOT NULL` | workflow 状态 |
| `definition_json` | `TEXT NOT NULL` | workflow 定义 JSON |
| `result_json` | `TEXT NOT NULL` | workflow 执行结果 JSON |
| `updated_at` | `TEXT NOT NULL` | 更新时间 |

### `quality_history`

质量治理历史表，通过 `record_type` 承载三类记录：`eval_run`、`trust_snapshot`、`degradation_decision`。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `INTEGER PRIMARY KEY AUTOINCREMENT` | 自增记录 ID |
| `record_type` | `TEXT NOT NULL` | 记录类型 |
| `record_id` | `TEXT NOT NULL DEFAULT ''` | eval run ID |
| `agent_id` | `TEXT NOT NULL DEFAULT ''` | Agent ID |
| `task_id` | `TEXT NOT NULL DEFAULT ''` | 任务 ID |
| `component` | `TEXT NOT NULL DEFAULT ''` | 质量组件 |
| `status` | `TEXT NOT NULL DEFAULT ''` | eval 状态 |
| `decision` | `TEXT NOT NULL DEFAULT ''` | trust/degradation 决策 |
| `reason` | `TEXT NOT NULL DEFAULT ''` | 原因说明 |
| `score` | `REAL NOT NULL DEFAULT 0` | 质量或信任分数 |
| `quality_drop` | `REAL NOT NULL DEFAULT 0` | 质量下降值 |
| `created_at` | `TEXT NOT NULL` | 创建时间 |

## 主要数据流

1. `tasks` 写入任务主记录，`task_locks` 控制调度并发。
2. 执行过程中写入 `runtime_events` 和 `audit_events`。
3. 执行完成后写入 `task_runs`，质量治理写入 `quality_history`。
4. 技能安装写入 `skills`，扫描结果写入 `skill_scan_findings`。
5. 学习进化产物写入 `capability_assets`，过程事件写入 `evolution_events`。
6. 工作流执行和等待状态写入 `workflow_states`，服务重启后可恢复等待中的 workflow。
7. TUI 会话写入 `agent_sessions` 与 `conversation_turns`，下一轮调用会读取最近 N 轮 turn 注入 prompt。
