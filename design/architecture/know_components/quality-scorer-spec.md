---
id: "spec-know-quality-scorer-052"
title: "QualityScorer 组件规范"
aliases: ["QualityScorer规范", "知识质量评分", "quality-scorer"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "quality", "score"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K52"
layer: "K5"
depends_on:
  - "K30"
optional_deps:
  - "K31"
  - "K33"
conflicts_with: []
required_by:
  - "K50"
  - "K60"
assembly_profiles:
  - standard
  - enterprise
---

# QualityScorer 组件规范

## 1. 组件定位

`QualityScorer` 为知识块和知识社区生成质量分，用于检索排序、审核优先级和健康报告。

## 2. 评分维度

| 维度 | 说明 |
|------|------|
| source_trust | 人类/高信任 Agent/低信任 Agent 来源差异 |
| freshness | 是否过期，最近是否被更新 |
| citation_degree | 被 WikiLink 或图边引用次数 |
| conflict_risk | 是否存在冲突或 ambiguous 边 |
| retrieval_success | 被检索后是否帮助任务成功 |
| content_completeness | 标题、摘要、来源、正文是否完整 |

## 3. 接口定义

```go
type QualityScorer interface {
    ScoreChunk(ctx context.Context, chunkID string) (QualityScore, error)
    ScoreCommunity(ctx context.Context, communityID string) (QualityScore, error)
    RecomputeBatch(ctx context.Context, companyID string, chunkIDs []string) error
}
```

## 4. 契约

- 分数范围 0~1。
- 评分要保存原因向量，不能只有一个数字。
- Watchlist 和 Conflicting 默认降低检索权重。
- QualityScore 不改变知识状态，只给治理和检索使用。

