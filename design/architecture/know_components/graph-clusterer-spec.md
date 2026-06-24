---
id: "spec-know-graph-clusterer-022"
title: "GraphClusterer 组件规范"
aliases: ["GraphClusterer规范", "图谱聚类器", "社区发现"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "cluster", "community", "graphify"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K22"
layer: "K2"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "K60"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

# GraphClusterer 组件规范

## 1. 组件定位

`GraphClusterer` 对知识图谱做社区发现，生成知识主题群、中心节点、cohesion 分数和低质量社区信号。它借鉴 Graphify 的 `cluster.py`。

## 2. 接口定义

```go
type GraphClusterer interface {
    Cluster(ctx context.Context, graph KnowledgeGraph, opts ClusterOptions) (ClusterResult, error)
    ScoreCohesion(ctx context.Context, graph KnowledgeGraph, communityID string) (float64, error)
}
```

## 3. 算法策略

| 优先级 | 算法 | 说明 |
|--------|------|------|
| 1 | Leiden | 质量优先，用于大图 |
| 2 | Louvain | Leiden 不可用时降级 |
| 3 | Connected Components | 无权重/小图兜底 |

超大社区（默认超过全图 25% 且 >=10 节点）执行二次 split；cohesion 过低的大社区也二次拆分。

## 4. 输出

| 字段 | 说明 |
|------|------|
| `communities` | community_id → node_ids |
| `cohesion_scores` | 社区内边密度 |
| `god_nodes` | 高连接核心节点 |
| `thin_communities` | 低节点数社区 |
| `low_cohesion_alerts` | 可疑边界不清社区 |

