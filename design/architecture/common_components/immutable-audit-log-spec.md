---
id: "spec-common-immutable-audit-log-X02"
title: "ImmutableAuditLog 组件规范"
aliases: ["ImmutableAuditLog规范", "不可变审计日志", "immutable-audit-log-spec"]
type: "spec"
category: "design/architecture/common_components"
tags: ["component-spec", "common", "audit", "append-only", "legion"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "index-common-components"

component_id: "X02"
layer: "X"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "A02"
  - "A54"
  - "A61"
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# ImmutableAuditLog 组件规范

## 1. 组件定位

`ImmutableAuditLog` 是跨 MaaS 与 Agent Engine 的**仅追加审计链**。它记录任务状态流转、权限决策、HardLoop 证据、信任分事件、进化事件等高风险操作。

本组件保证“可追溯”，不承担业务状态存储；业务表更新失败时不得只写审计成功。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
type ImmutableAuditLog interface {
    Append(ctx context.Context, entry AuditEntry) (AuditReceipt, error)
    AppendBatch(ctx context.Context, entries []AuditEntry) ([]AuditReceipt, error)
    VerifyChain(ctx context.Context, scope AuditScope) (VerifyResult, error)
}

type AuditEntry struct {
    ID          string
    Scope       string // task | agent | maas | evolution
    ActorID     string
    Action      string
    ResourceID  string
    PayloadJSON []byte
    PrevHash    string
    CreatedAt   time.Time
}

type AuditReceipt struct {
    EntryID     string
    ContentHash string
    ChainHash   string
    Sequence    int64
}
```

<!-- @end-section -->

<!-- @section: behavior -->
---

## 3. 行为契约

| 契约 | 说明 |
|------|------|
| 仅 INSERT | 禁止 UPDATE / DELETE，修正事件必须追加新记录 |
| 哈希链 | 每条记录包含 `PrevHash` 与当前 `ContentHash` |
| 幂等写入 | 相同 `ID` 重试返回原 receipt，不重复追加 |
| 事务边界 | 业务状态写入与审计写入应处于同一事务或使用 outbox 补偿 |
| 证据完整 | HardLoop / 审批 / 权限拒绝必须包含可复核证据 |

<!-- @end-section -->

<!-- @section: storage -->
---

## 4. 存储后端

| 后端 | 场景 | 要求 |
|------|------|------|
| PostgreSQL | 标准生产 | 表级禁止 UPDATE/DELETE；按 `scope`、`resource_id` 建索引 |
| ClickHouse | 高吞吐审计分析 | 异步批量写；保留 PostgreSQL 主链 |
| JSONL Export | 离线归档 | 只作为导出格式，不作为在线查询主存储 |
| Stdout Noop | 本地开发 | 仅用于 minimal profile，不满足合规 |

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[../agent_components/agent-coordinator-spec|AgentCoordinator 组件规范]]
- [[../agent_components/evolution-event-log-spec|EvolutionEventLog 组件规范]]
- [[../maas_components/audit-logger-spec|AuditLogger 组件规范]]

<!-- @end-section -->
