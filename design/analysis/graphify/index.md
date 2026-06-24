---
id: "analysis-graphify-index-001"
title: "Graphify 代码库分析索引"
aliases: ["graphify分析", "Graphify分析索引", "graphify-codebase-analysis"]
type: "index"
category: "design/analysis/graphify"
tags: ["graphify", "analysis", "knowledge-graph", "architecture", "python"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
source_path: "../../../../graphify"
---

# Graphify 代码库分析索引

本目录记录对 `graphify/` 代码库的阅读、理解与架构分析。Graphify 是一个面向 AI 编程助手的知识图谱生成工具：把代码、文档、论文、图片、音视频等项目语料转成可查询的图结构，并输出 `graph.json`、`GRAPH_REPORT.md`、`graph.html`、Obsidian/Wiki/MCP 等消费形态。

## 阅读顺序

| 顺序 | 文档 | 内容 |
|------|------|------|
| 1 | [01-overview.md](./01-overview.md) | 产品定位、能力边界、技术栈、关键结论 |
| 2 | [02-pipeline-architecture.md](./02-pipeline-architecture.md) | `detect → extract → build → cluster → analyze → report → export` 主流水线 |
| 3 | [03-module-analysis.md](./03-module-analysis.md) | 核心 Python 模块职责、输入输出、实现细节 |
| 4 | [04-cli-and-integrations.md](./04-cli-and-integrations.md) | CLI 命令、助手技能安装、hooks、MCP、watch/update |
| 5 | [05-security-and-testing.md](./05-security-and-testing.md) | 安全设计、隐私边界、测试覆盖与质量策略 |
| 6 | [06-design-insights-for-legion.md](./06-design-insights-for-legion.md) | 对 Legion / LLM Wiki / Agent 体系的借鉴点 |

## 核心代码入口

| 文件 | 角色 |
|------|------|
| `graphify/graphify/__main__.py` | CLI 总入口、平台安装、查询、增量、导出、headless extract |
| `graphify/graphify/detect.py` | 文件发现、类型分类、敏感文件跳过、增量 manifest |
| `graphify/graphify/extract.py` | 多语言 tree-sitter AST 提取、跨文件调用推断、并行与缓存 |
| `graphify/graphify/llm.py` | 文档/论文/图片的 LLM 语义提取后端 |
| `graphify/graphify/build.py` | extraction JSON → NetworkX 图、schema 兼容、去重、增量合并 |
| `graphify/graphify/cluster.py` | Leiden/Louvain 社区发现与 cohesion 评分 |
| `graphify/graphify/analyze.py` | God nodes、惊喜连接、建议问题、图 diff |
| `graphify/graphify/report.py` | 生成 `GRAPH_REPORT.md` |
| `graphify/graphify/export.py` | JSON/HTML/Obsidian/Wiki/SVG/GraphML/Neo4j 导出 |
| `graphify/graphify/serve.py` | MCP stdio server，暴露图查询工具 |
| `graphify/graphify/security.py` | URL、路径、标签安全校验 |

## 一句话架构

Graphify 的核心架构是：**用本地 AST 解析拿确定性结构，用 LLM 补语义关系，用 NetworkX 统一建图，再把图导出成适合人和 Agent 消费的多种视图**。
