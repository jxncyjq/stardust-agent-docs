---
id: "spec-know-vector-store-032"
title: "VectorStore 组件规范"
aliases: ["VectorStore规范", "知识向量库", "sqlite-vss"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "vector", "embedding", "sqlite-vss"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K32"
layer: "K3"
depends_on:
  - "K30"
  - "X01"
optional_deps: []
conflicts_with: []
required_by:
  - "K40"
  - "K51"
assembly_profiles:
  - standard
  - enterprise
---

# VectorStore 组件规范

## 1. 组件定位

`VectorStore` 提供知识块向量索引和相似度检索。默认使用 sqlite-vss，避免引入外部向量数据库；向量生成委托 `EmbeddingProvider`。

## 2. 接口定义

```go
type VectorStore interface {
    UpsertEmbedding(ctx context.Context, chunkID string, content string) error
    SearchSimilar(ctx context.Context, req VectorSearchRequest) ([]VectorHit, error)
    DeleteByChunkIDs(ctx context.Context, chunkIDs []string) error
}
```

## 3. 检索契约

| 项 | 默认 |
|----|------|
| TopK | 20 |
| 距离 | cosine |
| 过滤 | company_id + status=approved |
| fallback | EmbeddingProvider 缺失时返回空 |

## 4. 一致性规则

- `EmbeddingProvider.ModelID` 或维度变化时，新建索引版本。
- chunk 内容 hash 未变化时不得重复生成 embedding。
- 删除知识块时同步删除向量。
- 向量检索结果必须返回 chunk_id、score、source_path。

