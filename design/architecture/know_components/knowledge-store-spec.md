---
id: "spec-know-knowledge-store-030"
title: "KnowledgeStore 组件规范"
aliases: ["KnowledgeStore规范", "知识存储", "knowledge-store"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "storage", "sqlite", "governance"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K30"
layer: "K3"
depends_on:
  - "X02"
optional_deps: []
conflicts_with: []
required_by:
  - "K31"
  - "K32"
  - "K33"
  - "K50"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

# KnowledgeStore 组件规范

## 1. 组件定位

`KnowledgeStore` 是 LLM Wiki 的权威存储，管理知识块、图节点、图边、版本、状态和多租户隔离。底层默认 SQLite WAL，可在企业部署中替换为 PostgreSQL。

## 2. 核心表

### `knowledge_chunks`

| 字段 | 说明 |
|------|------|
| `id` | SHA-256 内容寻址 |
| `doc_id` | 文档 ID |
| `company_id` | 多租户隔离，所有查询必须过滤 |
| `repo_id` | 可选项目/仓库 ID |
| `path/title/content` | FTS 索引字段 |
| `embedding_ref` | 向量索引引用 |
| `created_by` | human_id / agent_id |
| `source_type` | human / agent_task / agent_learn / imported |
| `status` | pending_review / approved / archived / rejected 等 |
| `quality_score` | 0~1 |
| `version` | 版本号 |

### `knowledge_edges`

保存图谱边，字段含 source_id、target_id、relation、confidence、confidence_score、source_file、source_location。

### `knowledge_audit`

通过 `ImmutableAuditLog` 写入，只追加。

## 3. 接口定义

```go
type KnowledgeStore interface {
    PutChunk(ctx context.Context, chunk KnowledgeChunk) error
    GetChunk(ctx context.Context, id string) (KnowledgeChunk, error)
    UpdateStatus(ctx context.Context, id string, from, to KnowledgeStatus, reason string) error
    PutGraph(ctx context.Context, graph KnowledgeGraphSnapshot) error
    PruneBySource(ctx context.Context, companyID string, sources []string) error
}
```

## 4. 行为契约

- 所有读写必须带 `company_id`。
- 内容寻址相同且版本相同的写入幂等。
- 状态更新必须走合法状态机并写审计。
- Wiki DB 不存储 Agent evolution_assets，避免层依赖倒置。

