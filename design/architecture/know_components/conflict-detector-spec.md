---
id: "spec-know-conflict-detector-051"
title: "ConflictDetector 组件规范"
aliases: ["ConflictDetector规范", "知识冲突检测", "冲突检测器"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "conflict", "governance"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K51"
layer: "K5"
depends_on:
  - "K32"
  - "K33"
optional_deps:
  - "C70"
conflicts_with: []
required_by:
  - "K50"
assembly_profiles:
  - standard
  - enterprise
---

# ConflictDetector 组件规范

## 1. 组件定位

`ConflictDetector` 在新知识进入审核流程前检测其是否与已有 Approved 知识矛盾。

## 2. 两级检测

| 级别 | 机制 | 条件 |
|------|------|------|
| 语义相似冲突 | VectorStore TopK=10 | similarity > 0.85 后进入 LLM 轻量判断 |
| 显式引用冲突 | WikiLinkGraph 直接引用 | 同主题结论相反 |

## 3. 接口定义

```go
type ConflictDetector interface {
    Detect(ctx context.Context, candidate KnowledgeChunk) (ConflictResult, error)
}
```

## 4. LLM 判断

- 通过 `C70 MaasInferenceClient.Complete(PurposeConflictJudgment)` 使用 Light Tier。
- 输入只包含候选 chunk 与 TopK 相似 approved chunks。
- 输出结构化 JSON：`conflict: bool`、`reason`、`conflicting_chunk_ids`。
- LLM 判断只能把知识标记为 `Conflicting`，不能自动拒绝或批准。
- 不直接调用 `ModelRouter`、`ModelProvider` 或 provider SDK。

## 5. 输出

| 字段 | 说明 |
|------|------|
| `has_conflict` | 是否冲突 |
| `conflict_type` | semantic / wikilink / both |
| `evidence` | 相似度、引用路径、LLM reason |
| `severity` | low / medium / high |
