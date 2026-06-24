---
id: "spec-know-entity-deduplicator-021"
title: "EntityDeduplicator 组件规范"
aliases: ["EntityDeduplicator规范", "实体去重器", "知识实体合并"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "dedup", "entity", "graphify"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K21"
layer: "K2"
depends_on: []
optional_deps:
  - "C70"
conflicts_with: []
required_by:
  - "K20"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

# EntityDeduplicator 组件规范

## 1. 组件定位

`EntityDeduplicator` 合并同一知识实体的重复节点，避免图谱中出现 `ModelRouter`、`model_router`、`Model Router` 等多个等价节点。

## 2. 去重流水线

借鉴 Graphify `dedup.py`：

1. exact normalization：小写、去非字母数字。
2. entropy gate：低信息短标签不做 fuzzy merge。
3. MinHash/LSH blocking：找候选。
4. Jaro-Winkler verification。
5. same-community boost。
6. union-find merge。
7. 可选 LLM tiebreak。

## 3. 接口定义

```go
type EntityDeduplicator interface {
    Deduplicate(ctx context.Context, nodes []GraphNode, edges []GraphEdge, opts DedupOptions) (DedupResult, error)
}
```

## 4. 安全约束

- 禁止跨 `company_id` 去重。
- 禁止跨 repo 自动 fuzzy merge；跨项目只允许 exact alias 或人工确认。
- LLM tiebreak 只能输出 yes/no，不允许生成新知识。
- LLM tiebreak 通过 `C70 MaasInferenceClient.Complete(PurposeEntityDedup)` 执行，不直接调用 `ModelRouter` 或 provider SDK。
- 每次 merge 记录 remap 表，支持追溯和回滚。
