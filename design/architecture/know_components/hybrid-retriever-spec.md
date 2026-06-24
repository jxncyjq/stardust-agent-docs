---
id: "spec-know-hybrid-retriever-040"
title: "HybridRetriever 组件规范"
aliases: ["HybridRetriever规范", "混合检索", "三路检索"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "retrieval", "rrf", "hybrid"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K40"
layer: "K4"
depends_on:
  - "K31"
optional_deps:
  - "K32"
  - "K33"
conflicts_with: []
required_by:
  - "K41"
  - "K61"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

# HybridRetriever 组件规范

## 1. 组件定位

`HybridRetriever` 实现 LLM Wiki 的融合检索：全文是最小必需源，向量和图谱是可选增强源。启用 K32/K33 时形成“向量 + FTS5 + WikiLink”三路 RRF（Reciprocal Rank Fusion）排序，不引入 Neo4j 等外部图数据库。

## 2. 检索源

| 源 | 组件 | TopK | 说明 |
|----|------|------|------|
| vector | K32 VectorStore | 20 | 语义相似 |
| fts | K31 KnowledgeFtsEngine | 20 | 关键词/短语/BM25，必需 |
| graph | K33 WikiLinkGraph | BFS 2 跳 | 关联扩展 |

## 3. 接口定义

```go
type HybridRetriever interface {
    Retrieve(ctx context.Context, req RetrieveRequest) (RetrieveResult, error)
}
```

## 4. RRF 融合

```
score(doc) = Σ 1 / (k + rank_i(doc))
```

默认 `k=60`。同一 chunk 来自多路时合并 evidence，保留每路排名和原始分数。

## 5. 行为契约

- 三路检索并行执行。
- K31 必须可用；K32/K33 缺失或失败不导致整体失败，结果标记 `degraded_source`。
- minimal profile 下只启用 FTS 检索；standard/enterprise 启用向量与 WikiLink 增强。
- 默认只检索 `Approved` 知识。
- 返回结果必须包含 source、confidence、status、why_matched。
