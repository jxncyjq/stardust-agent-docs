---
id: "spec-agent-episodic-memory-store-042"
title: "EpisodicMemoryStore 组件规范"
aliases: ["EpisodicMemoryStore规范", "情景记忆库", "episodic-memory-store-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "memory", "episodic", "sqlite", "embedding"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A42"
layer: "L4"
depends_on:
  - "X01"
optional_deps: []
conflicts_with: []
required_by:
  - "A40"
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# EpisodicMemoryStore 组件规范

## 1. 组件定位

`EpisodicMemoryStore` 保存跨任务的情景记忆：任务摘要、工具调用轨迹、失败原因、人工反馈与压缩后的上下文快照。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
type EpisodicMemoryStore interface {
    Put(ctx context.Context, episode Episode) error
    Search(ctx context.Context, query EpisodeQuery) ([]EpisodeHit, error)
    Get(ctx context.Context, episodeID string) (Episode, error)
}
```

<!-- @end-section -->

<!-- @section: storage -->
---

## 3. 存储与检索

| 能力 | 设计 |
|------|------|
| 主存储 | SQLite WAL / PostgreSQL，按部署模式选择 |
| 全文检索 | FTS5，标题权重 5x |
| 向量检索 | EmbeddingProvider 生成向量，TopK=5 |
| 相似阈值 | 默认余弦相似度 > 0.7 |
| 数据保留 | 可按租户配置 TTL，安全事件不自动删除 |

<!-- @end-section -->

<!-- @section: contracts -->
---

## 4. 行为契约

- 任务失败也必须写入失败 episode。
- 写入前脱敏 PII 与密钥。
- 检索结果必须包含 `source_episode_id`，注入上下文可追溯。

<!-- @end-section -->

## 相关文档

- [[design-agent-tiered-memory-001|分层记忆注入设计]] — 扩展本组件的落地设计（Lane B episodic 检索 + 落盘 + hermes 注入策略移植）
