---
id: "spec-know-knowledge-graph-builder-020"
title: "KnowledgeGraphBuilder 组件规范"
aliases: ["KnowledgeGraphBuilder规范", "知识图谱构建器", "graph-builder"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "knowledge-graph", "builder", "graphify"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K20"
layer: "K2"
depends_on:
  - "K21"
optional_deps:
  - "X05"
conflicts_with: []
required_by:
  - "K22"
  - "K30"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

# KnowledgeGraphBuilder 组件规范

## 1. 组件定位

`KnowledgeGraphBuilder` 将 AST 和 LLM 的 extraction fragments 合并为统一知识图谱。它借鉴 Graphify `build.py` 的 ID 归一、schema 兼容、方向保留、增量合并和 pruning 机制。

## 2. 接口定义

```go
type KnowledgeGraphBuilder interface {
    Build(ctx context.Context, fragments []ExtractionFragment, opts BuildOptions) (KnowledgeGraph, error)
    BuildMerge(ctx context.Context, existing GraphSnapshot, fragments []ExtractionFragment, pruneSources []string) (KnowledgeGraph, error)
}
```

## 3. 构建步骤

1. 兼容 legacy schema：`links → edges`、`from/to → source/target`。
2. 规范化 node id 与 source path。
3. 跳过外部库 dangling edge。
4. 调用 `EntityDeduplicator`。
5. 合并 hyperedges。
6. 保留边方向 `_src/_tgt`，即使底层图用于无向社区检测。
7. 写入 `KnowledgeStore` 图快照和索引任务。

## 4. 行为契约

- 不静默缩小图；除非显式传入 `pruneSources`。
- 任何 `AMBIGUOUS` 边不得提升为 `EXTRACTED`。
- 节点和边必须保留 source_file/source_location。
- 多租户场景必须写入 `company_id` 与 `repo_id`。

