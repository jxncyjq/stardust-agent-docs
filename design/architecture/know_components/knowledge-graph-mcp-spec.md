---
id: "spec-know-knowledge-graph-mcp-061"
title: "KnowledgeGraphMCP 组件规范"
aliases: ["KnowledgeGraphMCP规范", "知识图谱MCP", "graph query mcp"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "mcp", "graph-query", "agent-tool"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K61"
layer: "K6"
depends_on:
  - "K40"
  - "X04"
  - "X05"
optional_deps: []
conflicts_with: []
required_by: []
assembly_profiles:
  - standard
  - enterprise
---

# KnowledgeGraphMCP 组件规范

## 1. 组件定位

`KnowledgeGraphMCP` 将知识图谱暴露为 Agent 可调用工具，借鉴 Graphify `serve.py`：关键词定位起点，再执行 BFS/DFS/最短路径/邻居查询。

## 2. 工具列表

| 工具 | 说明 |
|------|------|
| `query_graph` | 自然语言问题 → 相关子图文本 |
| `get_node` | 根据 label/id 查节点详情 |
| `get_neighbors` | 查直接邻居和边 |
| `shortest_path` | 查两个概念之间路径 |
| `get_community` | 查社区节点 |
| `search_knowledge` | 调用 HybridRetriever 做三路检索 |

## 3. 接口定义

```go
type KnowledgeGraphMCP interface {
    QueryGraph(ctx context.Context, req GraphQueryRequest) (GraphQueryResult, error)
    GetNode(ctx context.Context, req NodeRequest) (NodeDetail, error)
    GetNeighbors(ctx context.Context, req NeighborRequest) ([]NeighborEdge, error)
    ShortestPath(ctx context.Context, req PathRequest) (GraphPath, error)
}
```

## 4. 安全契约

- 图路径和导出路径必须通过 `PathGuard`。
- 输出所有标签和路径必须通过 `OutputSanitizer`。
- token_budget 默认 2000，硬上限 8000。
- 默认隐藏 PendingReview/Rejected 知识，除非调用者具备治理权限。

