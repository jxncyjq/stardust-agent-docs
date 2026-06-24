---
id: "spec-know-knowledge-fts-engine-031"
title: "KnowledgeFtsEngine 组件规范"
aliases: ["KnowledgeFtsEngine规范", "FTS5检索", "BM25检索"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "fts5", "bm25", "sqlite"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K31"
layer: "K3"
depends_on:
  - "K30"
optional_deps: []
conflicts_with: []
required_by:
  - "K40"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

# KnowledgeFtsEngine 组件规范

## 1. 组件定位

`KnowledgeFtsEngine` 提供 SQLite FTS5 全文检索能力，对应设计文档 §3.2b。它负责关键词、短语、NEAR、布尔查询和 snippet 高亮。

## 2. FTS5 表

```sql
CREATE VIRTUAL TABLE knowledge_fts USING fts5(
  path,
  title,
  content,
  tokenize = 'porter unicode61'
);
```

BM25 权重：

| 字段 | 权重 |
|------|------|
| title | 5.0 |
| path | 1.0 |
| content | 1.0 |

## 3. 查询语法

| 语法 | 示例 | 说明 |
|------|------|------|
| 简单词 | `legion` | 自动转 `legion*` |
| 多词 AND | `legion agent` | `legion* AND agent*` |
| 精确短语 | `"model routing"` | 原样短语 |
| NEAR | `NEAR(agent task, 5)` | 近邻 |
| OR/NOT | `agent OR skill NOT test` | 布尔 |
| 前缀 | `orche*` | 原样前缀 |

## 4. 接口定义

```go
type KnowledgeFtsEngine interface {
    Search(ctx context.Context, req FtsSearchRequest) ([]FtsHit, error)
    Reindex(ctx context.Context, companyID string, chunkIDs []string) error
    SanitizeQuery(raw string) (string, error)
}
```

## 5. 行为契约

- 普通用户查询必须经过 `SanitizeQuery`。
- Snippet 使用 `<mark>` 高亮，上下文默认 40 词。
- 只返回 `Approved`，除非调用者显式请求治理视图。
- 查询必须带 `company_id`。

