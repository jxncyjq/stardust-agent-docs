---
id: "analysis-graphify-module-analysis-001"
title: "Graphify 核心模块分析"
aliases: ["Graphify模块分析", "graphify modules"]
type: "analysis"
category: "design/analysis/graphify"
tags: ["graphify", "modules", "python", "implementation"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
related_docs:
  - id: "analysis-graphify-index-001"
    relation: "parent"
    path: "./index.md"
---

# Graphify 核心模块分析

## 1. `detect.py`：语料发现与分类

职责：

- 根据扩展名、shebang、论文特征、Google Workspace shortcut 判断文件类型。
- 跳过敏感文件，如 `.env`、密钥、证书、credential、token。
- 支持 `.graphifyignore` 与 `.graphifyinclude`。
- 统计 corpus 规模，给出“是否值得构图”的提示。
- 维护 `manifest.json`，支持增量扫描。

设计特点：

- 文件类型不是简单扩展名映射：Markdown 可能被论文特征提升为 `paper`。
- Google Workspace 是 opt-in，避免默认访问用户云文档。
- Office 转 Markdown 是转换层，不直接改变原文件。

## 2. `extract.py`：确定性结构提取核心

这是项目最大、最关键的模块。

职责：

- 使用 tree-sitter 对多语言代码做结构提取。
- 为每种语言配置 `LanguageConfig`，抽取 class/function/import/call。
- 处理 Python、JS/TS、Go、Rust、Java、C/C++、Ruby、C#、Kotlin、Scala、PHP、Swift、Lua、Zig、PowerShell、Elixir、Objective-C、Julia、Fortran、Dart、Verilog、SQL、Markdown 等。
- 处理 JS/TS 特殊场景：动态 import、CommonJS require、tsconfig paths、Svelte/Vite 文件解析。
- 二次解析跨文件关系：Python/Java import resolution、全语言 cross-file call resolution。
- 使用 `ProcessPoolExecutor` 并行 AST 提取，Windows 失败时降级顺序执行。
- 用 `cache.py` 避免重复提取。

重要实现点：

| 机制 | 说明 |
|------|------|
| `_make_id()` | 把路径、符号名归一为稳定 node id |
| `_safe_extract()` | 单文件失败不拖垮整个 corpus |
| `_DISPATCH` | 后缀到 extractor 的分发表 |
| raw_calls | 先记录未解析调用，合并所有节点后再推断跨文件 calls |
| import evidence | 有显式 import 证据时把跨文件 calls 从 `INFERRED` 提升为 `EXTRACTED` |
| ambiguity guard | 对同名多候选函数跳过推断，避免虚假 god nodes |

主要风险是模块过大。新增语言、修 bug、性能优化、安全处理都聚集在一个文件，长期会提高维护成本。

## 3. `llm.py`：语义提取后端

职责：

- 为非代码语料提供 headless LLM extraction。
- 支持 Claude、Kimi、Gemini、OpenAI、Ollama、Bedrock。
- 按 token 估算切 chunk，过长时自适应重试。
- 解析 LLM JSON，限制响应最大 10MB，避免 `json.loads` 内存风险。
- 估算 token 成本。

关键点：

- prompt 强制输出统一 schema。
- OpenAI-compatible 后端复用 `_call_openai_compat()`。
- Kimi reasoning 模型需要关闭 thinking，否则 content 可能为空。
- Ollama 支持本地语义提取，但质量依赖本地模型。

## 4. `build.py`：图构建与合并

职责：

- 把 extraction dict 转为 NetworkX `Graph` / `DiGraph`。
- 兼容旧 schema：`links → edges`、`from/to → source/target`。
- 对节点 `source_file` 做路径归一。
- 对边端点做 normalized ID remap，容忍 LLM 生成略有差异的 ID。
- 跳过指向外部库/stdlib 的 dangling edges。
- 合并多个 extraction，并运行实体去重。
- 支持增量合并、删除文件 pruning、global graph repo prefix。

设计亮点：

- 无向图也保存 `_src/_tgt`，避免展示时丢失原始方向。
- `build_merge()` 直接读 JSON，避免 NetworkX round-trip 翻转方向。
- 对“图无意缩小”有保护，除非显式 pruning 或 dedup。

## 5. `dedup.py`：实体去重

去重流水线：

1. 文本归一 exact match。
2. entropy gate 过滤低信息标签。
3. MinHash/LSH 找候选。
4. Jaro-Winkler 验证。
5. 同社区加分。
6. union-find 合并。
7. 可选 LLM tiebreak。

关键约束：global graph 中不同 repo 的节点禁止跨项目去重，避免把不同项目里的同名概念误合并。

## 6. `cluster.py`：社区发现

职责：

- 优先使用 graspologic Leiden。
- 缺失时降级 NetworkX Louvain。
- directed graph 先转 undirected。
- isolate 单独成社区。
- 超大社区二次 split。
- cohesion 过低的大社区再拆分。

社区 ID 按规模降序重排，保证运行结果更稳定。

## 7. `analyze.py` 与 `report.py`

`analyze.py` 提供：

- `god_nodes()`：排除文件 hub 和 concept node，找高连接真实实体。
- `surprising_connections()`：优先找跨文件、跨类型、跨目录、跨社区连接。
- `suggest_questions()`：根据 ambiguous edges、bridge nodes、inferred edges、isolated nodes、低 cohesion 社区生成问题。
- `graph_diff()`：比较两版图。

`report.py` 把这些结果渲染为 `GRAPH_REPORT.md`，面向人类和 Agent。报告包含 corpus check、summary、freshness、community hubs、god nodes、surprises、ambiguous edges、knowledge gaps、suggested questions。

## 8. `export.py`、`wiki.py`、`serve.py`

导出层把同一张图变成多种消费形态：

| 模块 | 输出 |
|------|------|
| `export.to_json()` | `graph.json` |
| `export.to_html()` | vis-network 交互 HTML |
| `export.to_obsidian()` | Obsidian vault |
| `wiki.to_wiki()` | Agent 可爬取 Markdown Wiki |
| `export.to_svg()` | SVG 静态图 |
| `export.to_graphml()` | Gephi/yEd |
| `export.to_cypher()` / `push_to_neo4j()` | Neo4j |
| `serve.py` | MCP stdio 工具：query/get_node/get_neighbors/get_community/shortest_path |

`serve.py` 的查询不是 LLM 语义搜索，而是关键词匹配起点 + BFS/DFS 图遍历，并支持根据问题词推断 edge context filter。

