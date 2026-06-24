---
id: "spec-component-semantic-cache-016"
title: "SemanticCache 组件规范"
aliases: ["SemanticCache规范", "语义缓存", "semantic-cache-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "cache", "pipeline", "cost-saving", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C16"
layer: "L2"
depends_on: []
optional_deps:
  - "X01"   # EmbeddingProvider — 可复用平台统一嵌入端口；缺失时使用 MaaS 内部嵌入适配器或退化为 miss
conflicts_with: []
required_by: []
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# SemanticCache 组件规范

## 1. 组件定位

`SemanticCache` 在路由决策前拦截请求，检查是否存在**语义相似的历史响应**，若命中则直接返回缓存结果，跳过上游 AI 调用，降低成本和延迟。

```
QuotaChecker → SemanticCache → ModelRouter → ProviderExecutor
                   │
                   ├── 缓存命中 → 直接返回历史响应（跳过后续节点）
                   └── 缓存未命中 → 继续，完成后异步存入缓存
```

**Noop 行为**：C16 未注册时，框架注入 `NoopSemanticCache`，始终缓存未命中，透传上游。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// SemanticCache 提供语义相似度缓存查询和存储。
// 并发安全：所有方法可被多个 goroutine 并发调用。
type SemanticCache interface {
    // Lookup 查找语义相似的缓存响应。
    // 返回 (nil, nil) 表示缓存未命中（正常情况，不是错误）。
    // 返回 (*CacheHit, nil) 表示命中，调用方直接返回缓存响应。
    Lookup(ctx context.Context, input *ModelInput) (*CacheHit, error)

    // Store 异步存储请求+响应到缓存（不阻塞响应返回）。
    Store(ctx context.Context, input *ModelInput, output *ModelOutput)
}

// CacheHit 缓存命中结果。
type CacheHit struct {
    Output      *ModelOutput
    Similarity  float64    // 语义相似度分数 [0, 1]
    CachedAt    time.Time
    TTL         time.Duration
}
```

<!-- @end-section -->

<!-- @section: implementation -->
---

## 3. 实现思路

```
语义缓存的核心是向量相似度搜索：

1. Lookup：
   a. 将 ModelInput.Messages 编码为向量（优先调用 X01 EmbeddingProvider）
   b. 在向量数据库（如 pgvector / Redis）中搜索 top-K 最近邻
   c. 相似度 >= Threshold → 返回对应的历史响应
   d. 相似度 < Threshold → 缓存未命中

2. Store（异步）：
   a. 编码 ModelInput 为向量
   b. 存储 (向量, ModelInput, ModelOutput) 到向量数据库
   c. 设置 TTL
```

<!-- @end-section -->

<!-- @section: config -->
---

## 4. 配置 Schema

```go
// SemanticCacheConfig 是 SemanticCache 的配置。
type SemanticCacheConfig struct {
    // SimilarityThreshold 命中阈值 [0, 1]，低于此值视为未命中。
    SimilarityThreshold float64 `default:"0.95" validate:"min=0.5,max=1"`

    // TTL 缓存条目的生存时间。
    TTL time.Duration `default:"24h"`

    // Backend 向量存储后端。
    Backend string `default:"pgvector" validate:"oneof=pgvector redis_vsm memory"`

    // EmbeddingModel 用于计算语义向量的嵌入模型 ID。
    EmbeddingModel string `default:"text-embedding-3-small"`

    // MaxCacheSize 最大缓存条目数（超出时 LRU 淘汰）。
    MaxCacheSize int `default:"100000"`

    // ExcludeModels 不进行缓存的模型列表（如带 temperature>0 的随机性模型）。
    ExcludeModels []string

    // OnlyStream=false 才缓存（流式请求通常不缓存）
    CacheStreamResponses bool `default:"false"`
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 5. 行为契约

| 契约 | 说明 |
|------|------|
| **Lookup 失败降级** | 嵌入模型或向量库不可用时，返回未命中（不报错），保证请求继续 |
| **Store 异步** | Store 立即返回，写入在后台执行，失败静默 |
| **temperature>0 不缓存** | 有随机性的请求不参与缓存查询和存储 |
| **流式请求默认不缓存** | CacheStreamResponses=false 时跳过 |
| **Noop 始终未命中** | C16 未注册时的 Noop 实现透传所有请求 |

<!-- @end-section -->

<!-- @section: checklist -->
---

## 6. 实现检查清单

```
SemanticCache
  ☐ Lookup：嵌入编码 + 向量相似度搜索
  ☐ Lookup 失败时返回未命中（降级，不报错）
  ☐ Store：异步执行，失败静默
  ☐ temperature>0 时跳过
  ☐ CacheStreamResponses=false 时跳过流式请求

Noop 实现
  ☐ Lookup 始终返回 (nil, nil)
  ☐ Store 为无操作
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C16 在依赖图中的位置）
- model-provider-spec.md（C01，ProviderWithEmbedding 接口用于向量编码）

<!-- @end-section -->
