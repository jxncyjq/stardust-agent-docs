---
id: "spec-know-wiki-report-generator-060"
title: "WikiReportGenerator 组件规范"
aliases: ["WikiReportGenerator规范", "知识图谱报告", "GRAPH_REPORT"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "report", "graphify", "agent-context"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K60"
layer: "K6"
depends_on:
  - "K22"
  - "X05"
optional_deps:
  - "K52"
conflicts_with: []
required_by: []
assembly_profiles:
  - standard
  - enterprise
---

# WikiReportGenerator 组件规范

## 1. 组件定位

`WikiReportGenerator` 生成 Agent 首读的知识图谱报告，借鉴 Graphify 的 `GRAPH_REPORT.md`：God Nodes、Surprising Connections、Communities、Ambiguous Edges、Knowledge Gaps、Suggested Questions。

## 2. 报告结构

| 章节 | 说明 |
|------|------|
| Corpus Check | 文件数、知识块数、图谱新鲜度 |
| Summary | 节点/边/社区/置信度统计 |
| Graph Freshness | 构图版本、source snapshot |
| Community Hubs | 社区导航 |
| God Nodes | 核心高连接概念 |
| Surprising Connections | 跨社区/跨类型/低置信度连接 |
| Ambiguous Edges | 需要复核的边 |
| Knowledge Gaps | 孤立节点、缺失 WikiLink、薄社区 |
| Suggested Questions | 推荐 Agent/人类追问 |

## 3. 接口定义

```go
type WikiReportGenerator interface {
    Generate(ctx context.Context, req ReportRequest) (WikiReport, error)
}
```

## 4. 输出安全

- 所有标签、路径、snippet 经 `OutputSanitizer`。
- Markdown 输出不得包含未转义 HTML 控制片段。
- 报告中每个问题必须带 reason，避免空泛建议。

## 5. Agent 使用方式

`CognitiveCore` 在回答代码库/架构/知识问题前，应优先读取最新 `WikiReport`，再决定是否调用 `KnowledgeGraphMCP` 或读取原文。

