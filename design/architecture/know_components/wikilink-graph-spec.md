---
id: "spec-know-wikilink-graph-033"
title: "WikiLinkGraph 组件规范"
aliases: ["WikiLinkGraph规范", "WikiLink图谱", "知识链接图"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "wikilink", "graph", "sqlite"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K33"
layer: "K3"
depends_on:
  - "K30"
optional_deps: []
conflicts_with: []
required_by:
  - "K40"
  - "K51"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

# WikiLinkGraph 组件规范

## 1. 组件定位

`WikiLinkGraph` 维护 `[[目标]]` / `[[目标|显示文本]]` 构成的知识链接图。图谱持久化为 SQLite 邻接表，支持 gap-detect、hub 识别和 BFS 2 跳扩展。

## 2. 数据表

| 字段 | 说明 |
|------|------|
| `company_id` | 租户隔离 |
| `source_path` | 源文档路径 |
| `target_path` | 目标文档路径 |
| `display_text` | 可选显示文本 |
| `updated_at` | 更新时间 |

`target_path` 必须建立索引，支持反向查找。

## 3. 接口定义

```go
type WikiLinkGraph interface {
    ExtractLinks(content string) []WikiLink
    UpsertLinks(ctx context.Context, source string, targets []WikiLink) error
    DetectGaps(ctx context.Context, companyID string) ([]KnowledgeGap, error)
    FindHubs(ctx context.Context, companyID string) ([]WikiHub, error)
    BFSNeighbors(ctx context.Context, roots []string, depth int) ([]WikiNode, error)
}
```

## 4. 行为契约

- 保存文件时先删除 source 旧出链，再插入新出链。
- BFS 使用 SQL recursive CTE，限制最大深度默认 2，硬上限 4。
- Gap 按被引用次数排序。
- `target_path` 解析必须归一化，避免大小写/后缀差异造成重复。

