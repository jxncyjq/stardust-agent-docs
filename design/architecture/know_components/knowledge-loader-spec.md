---
id: "spec-know-knowledge-loader-041"
title: "KnowledgeLoader 组件规范"
aliases: ["KnowledgeLoader规范", "渐进式知识加载", "Agent知识加载"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "loader", "agent-context"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K41"
layer: "K4"
depends_on:
  - "K40"
optional_deps: []
conflicts_with: []
required_by: []
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

# KnowledgeLoader 组件规范

## 1. 组件定位

`KnowledgeLoader` 为 Agent 提供两阶段渐进式知识加载，避免初始化时把知识库全文塞入上下文。

## 2. 两阶段加载

| 阶段 | 内容 | 触发 |
|------|------|------|
| Stage 1 | 元数据索引：标题、路径、摘要、触发条件 | Agent 初始化或任务开始 |
| Stage 2 | 完整内容：chunk 正文、引用、邻接节点 | Agent 明确需要或 HybridRetriever 命中 |

## 3. 接口定义

```go
type KnowledgeLoader interface {
    LoadIndex(ctx context.Context, req KnowledgeIndexRequest) (KnowledgeIndex, error)
    LoadFull(ctx context.Context, req KnowledgeLoadRequest) (KnowledgeBundle, error)
}
```

## 4. Agent 集成

- `CognitiveCore` 在 P5 位置注入 Stage 1。
- `AgentRuntime` 可通过工具调用触发 Stage 2。
- 每次注入必须记录 chunk_id 和检索 query。
- token 预算由调用方传入，Loader 负责裁剪与排序。

## 5. 缓存规则

- 元数据索引按 company_id + agent_role 缓存。
- 完整内容按 chunk_id 缓存。
- 知识状态从 Approved 变为 Archived/Rejected 时立即失效。

