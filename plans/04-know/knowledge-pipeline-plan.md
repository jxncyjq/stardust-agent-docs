---
id: "plans-know-knowledge-pipeline-001"
title: "Know 知识流水线计划"
type: "plan"
category: "plans/know"
tags: ["plan", "know", "pipeline", "retrieval"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# Know 知识流水线计划

## 目标

把代码、文档、人类 Wiki、Agent 产出加工为可检索、可治理、可追溯的知识图谱。

## 阶段

| 阶段 | 组件 | 交付 |
|------|------|------|
| K1 本地语料 | K10/K12/X04 | repo 扫描、代码结构抽取 |
| K2 图谱构建 | K20/K21/K22/X05 | 节点、边、去重、聚类 |
| K3 存储索引 | K30/K31/X02 | chunk store、FTS5、审计 |
| K4 混合检索与查询接口 | K32/K33/K40/K61-lite/X01 | 向量、WikiLink、RRF 融合、search_knowledge/get_node |
| K5 语义增强 | K13/C70/C17 | LLM 节点关系抽取 |
| K6 接口输出 | K60/K61-full/X05 | 报告、完整 MCP 工具 |

## MVP 验收

- minimal 下不依赖 LLM，仍可从代码和 Markdown 构建 FTS 可检索知识。
- standard 下启用向量和 WikiLink 增强检索。
- 每条知识保留 source、version、status、confidence。
- P3 必须提供 K61-lite，让 Agent 能通过 `search_knowledge` 和 `get_node` 查询已批准知识并带引用返回。
