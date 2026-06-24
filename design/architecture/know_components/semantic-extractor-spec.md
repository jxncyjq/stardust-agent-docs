---
id: "spec-know-semantic-extractor-013"
title: "SemanticExtractor 组件规范"
aliases: ["SemanticExtractor规范", "语义提取器", "LLM知识提取"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "semantic", "llm", "graphify"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K13"
layer: "K1"
depends_on:
  - "X05"
optional_deps:
  - "C70"
  - "C17"
conflicts_with: []
required_by:
  - "K20"
assembly_profiles:
  - standard
  - enterprise
---

# SemanticExtractor 组件规范

## 1. 组件定位

`SemanticExtractor` 通过 `C70 MaasInferenceClient` 调用 MaaS，从文档、论文、图片、转录稿中抽取知识节点、关系和 group relationship。它对应 Graphify 的 `llm.py` 与 skill 子代理语义提取阶段。

## 2. 接口定义

```go
type SemanticExtractor interface {
    ExtractSemantic(ctx context.Context, files []Path, opts SemanticExtractOptions) (ExtractionFragment, error)
}
```

## 3. 输出 Schema

输出必须符合统一 extraction schema：

```json
{
  "nodes": [],
  "edges": [],
  "hyperedges": [],
  "input_tokens": 0,
  "output_tokens": 0
}
```

每条边必须带 `confidence`：

| confidence | 含义 |
|------------|------|
| EXTRACTED | 文档显式声明、引用、标题结构 |
| INFERRED | LLM 归纳的概念关系 |
| AMBIGUOUS | 不确定，进入治理复核 |

## 4. MaaS 集成

- 默认使用 Standard Tier；大文档归纳可升 Advanced。
- 通过 `C70 MaasInferenceClient.Complete(PurposeSemanticExtract)` 请求推理；不直接调用 `ModelRouter`、`ModelProvider` 或 provider SDK。
- 可选 `ContentFilter` 对输入/输出做 PII 和安全过滤。
- token 成本写入 `CostAttributor` 或 Wiki 运行指标。

## 5. 安全与稳定性

- LLM 响应最大字节数设硬上限。
- JSON 解析失败返回空 fragment 并记录告警，不中断全量构图。
- 每个 chunk 记录 source_file 与 chunk_id，便于追溯。
- prompt 必须明确“只输出 JSON”，但实现仍需 schema repair。
