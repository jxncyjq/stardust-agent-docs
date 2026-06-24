---
id: "plans-agent-data-model-001"
title: "Agent 数据模型计划"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "data-model"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# Agent 数据模型计划

## MVP 表

| 表 | 用途 |
|----|------|
| `companies` | Company-lite |
| `agents` | Agent-lite 身份、角色、状态和策略 |
| `tasks` | 任务状态机 |
| `task_locks` | 原子锁定和过期回收 |
| `task_runs` | 单次执行记录 |
| `tool_calls` | 工具调用记录 |
| `approval_tickets` | A62-lite 工单 |
| `audit_events` | 不可变审计事件 |
| `memory_entries` | A40-A42 记忆条目 |
| `workflow_runs` | A70 工作流实例 |
| `workflow_nodes` | A70 工作流节点状态 |

## 状态机

### TaskStatus

```text
pending -> assigned -> running -> quality_review -> done
running -> suspended
running -> failed
suspended -> running
quality_review -> failed
```

### ApprovalStatus

```text
open -> approved
open -> denied
open -> canceled
```

## 数据约束

- 所有业务表必须带 `company_id`。
- 所有任务必须带 `agent_id`。
- `task_locks.task_id` 必须唯一。
- `audit_events` 只追加，不更新，不删除。
- `approval_tickets.subject_id` 指向 task 或 knowledge draft。

## P4/P5 扩展表

| 表 | 阶段 | 用途 |
|----|------|------|
| `trust_scores` | P4 | A61 信任分缓存和冷却期 |
| `skills` | P5 | A30/A32 技能元数据 |
| `skill_scan_findings` | P5 | A31 安全扫描结果 |
| `capability_assets` | P5 | A43 Gene/Capsule 能力资产 |
| `evolution_events` | P5 | A54 进化事件审计 |

## WorkflowStatus

```text
created -> running -> completed
running -> waiting_approval
running -> failed
waiting_approval -> running
waiting_approval -> canceled
failed -> compensated
```

## 存储策略

P2 阶段先提供：

- `memory` repository：用于单元测试和本地 fake run。
- `sqlite` repository：用于本地演示。

生产数据库适配可在 P4 后补。
