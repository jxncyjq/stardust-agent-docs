---
id: "agent-data-retention-001"
title: "Legion Agent 数据保留与导出"
type: "guide"
category: "backend/agent"
tags: ["agent", "data", "retention", "export"]
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

# Legion Agent 数据保留与导出

P11 为 SQLite 持久化数据补充了最小治理入口，覆盖审计事件、运行事件和质量历史三类数据。该能力用于运维清理、合规留存、质量趋势归档前检查，以及事故排查时导出可读快照。

## 范围

| 数据 | SQLite 表 | Retention 参数 | 说明 |
|------|-----------|----------------|------|
| 审计事件 | `audit_events` | `--audit-days` | 删除早于指定天数的审计记录 |
| 运行事件 | `runtime_events` | `--runtime-days` | 删除早于指定天数的运行事件 |
| 质量历史 | `quality_history` | `--quality-days` | 删除早于指定天数的质量评估历史 |

天数参数为 `0` 时表示跳过该类数据。默认执行 dry-run，只统计候选删除数量，不修改数据库。

## Retention 命令

```powershell
agent data retention --config agent.json --quality-days 30
agent data retention --config agent.json --quality-days 30 --apply
agent data retention --config agent.json --audit-days 180 --runtime-days 30 --quality-days 90 --apply
```

输出为稳定键值格式，便于脚本和 CI 采集：

```text
retention dry_run=false audit_events_deleted=0 runtime_events_deleted=0 quality_history_deleted=1
```

`--apply` 成功后会写入一条审计记录：

```text
action=storage.retention.apply
subject_type=storage
subject_id=sqlite
```

## 导出命令

```powershell
agent data export --config agent.json --out agent-export.json
```

当前导出快照包含：

- `audit_events`
- `runtime_events`

导出文件使用 JSON 格式，权限为 `0600`。命令输出示例：

```text
export=agent-export.json audit_events=3 runtime_events=12
```

## 安全与操作建议

- 生产环境先执行 dry-run，确认候选删除数量后再使用 `--apply`。
- 执行 `--apply` 前建议先使用 `agent backup --config agent.json --out agent.db.bak`。
- 不要把导出文件提交到仓库，导出中可能包含运行事件消息和审计主体信息。
- 数据保留仅支持 SQLite；非 SQLite 配置会直接失败，避免误以为远端存储已被清理。

