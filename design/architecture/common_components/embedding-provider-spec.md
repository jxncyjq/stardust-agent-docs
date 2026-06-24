---
id: "spec-common-embedding-provider-X01"
title: "EmbeddingProvider 组件规范"
aliases: ["EmbeddingProvider规范", "向量嵌入服务", "embedding-provider-spec"]
type: "spec"
category: "design/architecture/common_components"
tags: ["component-spec", "common", "embedding", "memory", "semantic-cache", "legion"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "index-common-components"

component_id: "X01"
layer: "X"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "A42"
  - "C16"
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# EmbeddingProvider 组件规范

## 1. 组件定位

`EmbeddingProvider` 是 Agent 记忆系统、LLM Wiki 与 MaaS 语义缓存共用的**向量嵌入抽象**，负责把文本、知识片段、任务轨迹摘要转为统一维度的向量。

它只提供向量生成能力，不负责存储、索引或相似度检索；检索由 `EpisodicMemoryStore`、`SemanticCache`、Wiki 检索引擎分别实现。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
type EmbeddingProvider interface {
    Embed(ctx context.Context, input EmbedInput) (EmbedResult, error)
    EmbedBatch(ctx context.Context, inputs []EmbedInput) ([]EmbedResult, error)
    ModelInfo() EmbeddingModelInfo
    Health(ctx context.Context) HealthStatus
}

type EmbedInput struct {
    Text       string
    Purpose    EmbedPurpose // memory | semantic_cache | wiki
    TenantID   string
    TraceID    string
}

type EmbedResult struct {
    Vector     []float32
    Dimension  int
    ModelID    string
    TokenUsage int
    Hash       string // normalized text SHA-256，用于去重
}
```

<!-- @end-section -->

<!-- @section: behavior -->
---

## 3. 行为契约

| 契约 | 说明 |
|------|------|
| 维度稳定 | 同一 `ModelID` 的 `Dimension` 必须稳定，变更模型需新建索引版本 |
| 批量优先 | 后台任务应优先使用 `EmbedBatch`，降低调用开销 |
| 输入归一化 | 生成 `Hash` 前执行空白归一化，但不得改写原始文本 |
| 可降级 | 缺失时返回零向量，调用方退化为 FTS5 / BM25 |
| 不写业务表 | 本组件不得写记忆、缓存或知识库表 |

<!-- @end-section -->

<!-- @section: errors -->
---

## 4. 错误分类

| 错误 | 恢复动作 |
|------|----------|
| `ErrInputTooLarge` | 调用方切分文本后重试 |
| `ErrRateLimited` | 指数退避或交给 MaaS RetryScheduler |
| `ErrModelChanged` | 阻止写入旧索引，触发重建 |
| `ErrProviderUnavailable` | 降级为全文检索 |

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[../agent_components/episodic-memory-store-spec|EpisodicMemoryStore 组件规范]]
- [[../agent_components/memory-provider-spec|MemoryProvider 组件规范]]
- [[../maas_components/semantic-cache-spec|SemanticCache 组件规范]]

<!-- @end-section -->
