---
id: "index-know-components"
title: "Knowledge Graph / LLM Wiki 组件规范索引"
aliases: ["know组件索引", "LLM Wiki组件索引", "知识图谱组件索引", "know-components-index"]
type: "index"
category: "design/architecture/know_components"
tags: ["index", "component-spec", "llm-wiki", "knowledge-graph", "graphify", "legion"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
related_docs:
  - id: "architecture-agent-engine-design-nocode-001"
    relation: "derived_from"
    path: "../agent-engine-design-nocode.md"
  - id: "analysis-graphify-index-001"
    relation: "informed_by"
    path: "../../analysis/graphify/index.md"
  - id: "index-components"
    relation: "sibling"
    path: "../maas_components/index.md"
---

# Knowledge Graph / LLM Wiki 组件规范索引

本目录收录 Legion **LLM Wiki / Knowledge Graph 引擎**的组件规格。该层负责把代码、文档、Agent 产出知识与人工知识加工为可检索、可治理、可追溯的知识图谱。

设计来源：

- `agent-engine-design-nocode.md` §3：LLM Wiki 知识库引擎设计
- `docs/design/analysis/graphify/`：Graphify 的 detect/extract/build/cluster/report/export/MCP 设计分析
- `maas_components`：组件规格格式、依赖字段、Noop 降级和装配方案写法

全平台 A/C/K/X 跨模块依赖请查阅：
→ **[[../platform-component-registry|平台组件注册表与跨模块依赖]]**

## 层次概览

```
K0  框架与注册层      — KnowComponentRegistry
K1  语料接入层        — CorpusDetector / ContentIngestor / CodeStructureExtractor / SemanticExtractor
K2  图谱构建层        — KnowledgeGraphBuilder / EntityDeduplicator / GraphClusterer
K3  存储索引层        — KnowledgeStore / KnowledgeFtsEngine / VectorStore / WikiLinkGraph
K4  检索加载层        — HybridRetriever / KnowledgeLoader
K5  治理质量层        — KnowledgeGovernance / ConflictDetector / QualityScorer
K6  报告与接口层      — WikiReportGenerator / KnowledgeGraphMCP
```

## 组件列表

### K0 框架与注册层

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| K00 | KnowComponentRegistry | [know-component-registry.md](./know-component-registry.md) | 组件依赖主表、装配方案、Noop 行为 |

### K1 语料接入层

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| K10 | CorpusDetector | [corpus-detector-spec.md](./corpus-detector-spec.md) | 文件发现、类型分类、敏感文件跳过、增量 manifest |
| K11 | ContentIngestor | [content-ingestor-spec.md](./content-ingestor-spec.md) | URL/Office/Google Workspace/音视频转 Markdown sidecar |
| K12 | CodeStructureExtractor | [code-structure-extractor-spec.md](./code-structure-extractor-spec.md) | 本地 AST 提取代码结构、导入、调用、数据库关系 |
| K13 | SemanticExtractor | [semantic-extractor-spec.md](./semantic-extractor-spec.md) | LLM 语义提取文档/论文/图片知识片段和关系 |

### K2 图谱构建层

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| K20 | KnowledgeGraphBuilder | [knowledge-graph-builder-spec.md](./knowledge-graph-builder-spec.md) | extraction fragments → 统一知识图谱，保留置信度和来源 |
| K21 | EntityDeduplicator | [entity-deduplicator-spec.md](./entity-deduplicator-spec.md) | exact/MinHash/Jaro-Winkler/可选 LLM 实体去重 |
| K22 | GraphClusterer | [graph-clusterer-spec.md](./graph-clusterer-spec.md) | Leiden/Louvain 社区发现、cohesion 评分、中心节点识别 |

### K3 存储索引层

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| K30 | KnowledgeStore | [knowledge-store-spec.md](./knowledge-store-spec.md) | `knowledge_chunks` 内容寻址、多租户隔离、版本状态 |
| K31 | KnowledgeFtsEngine | [knowledge-fts-engine-spec.md](./knowledge-fts-engine-spec.md) | SQLite FTS5 + BM25 标题权重 5x + snippet |
| K32 | VectorStore | [vector-store-spec.md](./vector-store-spec.md) | sqlite-vss 向量检索，依赖 EmbeddingProvider |
| K33 | WikiLinkGraph | [wikilink-graph-spec.md](./wikilink-graph-spec.md) | `[[WikiLink]]` 邻接表、gap detect、BFS |

### K4 检索加载层

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| K40 | HybridRetriever | [hybrid-retriever-spec.md](./hybrid-retriever-spec.md) | FTS5 必需，向量 + WikiLink 可选增强的 RRF 融合检索 |
| K41 | KnowledgeLoader | [knowledge-loader-spec.md](./knowledge-loader-spec.md) | 两阶段渐进式加载，先元数据后正文 |

### K5 治理质量层

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| K50 | KnowledgeGovernance | [knowledge-governance-spec.md](./knowledge-governance-spec.md) | 状态机、审核、版本、审计、裁决 |
| K51 | ConflictDetector | [conflict-detector-spec.md](./conflict-detector-spec.md) | 语义相似冲突 + 显式引用冲突 |
| K52 | QualityScorer | [quality-scorer-spec.md](./quality-scorer-spec.md) | 新鲜度、来源可信度、引用度、冲突度评分 |

### K6 报告与接口层

| ID | 组件 | 规范文档 | 说明 |
|----|------|----------|------|
| K60 | WikiReportGenerator | [wiki-report-generator-spec.md](./wiki-report-generator-spec.md) | Graphify 风格 God Nodes / Surprises / Gaps 报告 |
| K61 | KnowledgeGraphMCP | [knowledge-graph-mcp-spec.md](./knowledge-graph-mcp-spec.md) | query_graph/get_node/path/neighbor MCP 工具 |

## 公共组件

以下组件放在 `common_components`，供 Know / Agent / MaaS 等层复用：

| ID | 组件 | 规范文档 | 使用方 |
|----|------|----------|--------|
| X00 | EventBus | [../common_components/event-bus-spec.md](../common_components/event-bus-spec.md) | KnowledgeGovernance 治理事件、WikiReportGenerator 报告刷新通知 |
| X01 | EmbeddingProvider | [../common_components/embedding-provider-spec.md](../common_components/embedding-provider-spec.md) | VectorStore、SemanticCache、EpisodicMemoryStore |
| X02 | ImmutableAuditLog | [../common_components/immutable-audit-log-spec.md](../common_components/immutable-audit-log-spec.md) | KnowledgeGovernance、AgentCoordinator、EvolutionEventLog |
| X03 | SafeFetcher | [../common_components/safe-fetcher-spec.md](../common_components/safe-fetcher-spec.md) | ContentIngestor、外部 URL 接入 |
| X04 | PathGuard | [../common_components/path-guard-spec.md](../common_components/path-guard-spec.md) | CorpusDetector、KnowledgeGraphMCP、Export |
| X05 | OutputSanitizer | [../common_components/output-sanitizer-spec.md](../common_components/output-sanitizer-spec.md) | WikiReportGenerator、KnowledgeGraphMCP、HTML/YAML 导出 |

## MaaS 推理端口

| ID | 组件 | 规范文档 | 使用方 |
|----|------|----------|--------|
| C70 | MaasInferenceClient | [../maas_components/maas-inference-client-spec.md](../maas_components/maas-inference-client-spec.md) | SemanticExtractor、EntityDeduplicator、ConflictDetector 的 LLM 调用 |

## 快速入口

| 需求 | 文档 |
|------|------|
| 了解完整组件依赖 | [know-component-registry.md](./know-component-registry.md) |
| 设计知识加工流水线 | [corpus-detector-spec.md](./corpus-detector-spec.md) · [code-structure-extractor-spec.md](./code-structure-extractor-spec.md) · [semantic-extractor-spec.md](./semantic-extractor-spec.md) |
| 设计知识图谱构建 | [knowledge-graph-builder-spec.md](./knowledge-graph-builder-spec.md) · [entity-deduplicator-spec.md](./entity-deduplicator-spec.md) · [graph-clusterer-spec.md](./graph-clusterer-spec.md) |
| 设计三路检索 | [knowledge-fts-engine-spec.md](./knowledge-fts-engine-spec.md) · [vector-store-spec.md](./vector-store-spec.md) · [wikilink-graph-spec.md](./wikilink-graph-spec.md) · [hybrid-retriever-spec.md](./hybrid-retriever-spec.md) |
| 设计知识治理 | [knowledge-governance-spec.md](./knowledge-governance-spec.md) · [conflict-detector-spec.md](./conflict-detector-spec.md) |
| 给 Agent 使用知识图谱 | [knowledge-loader-spec.md](./knowledge-loader-spec.md) · [knowledge-graph-mcp-spec.md](./knowledge-graph-mcp-spec.md) |
