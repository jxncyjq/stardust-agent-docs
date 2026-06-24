---
id: "spec-know-code-structure-extractor-012"
title: "CodeStructureExtractor 组件规范"
aliases: ["CodeStructureExtractor规范", "代码结构提取器", "AST提取器"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "code", "ast", "tree-sitter", "graphify"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K12"
layer: "K1"
depends_on:
  - "X04"
optional_deps: []
conflicts_with: []
required_by:
  - "K20"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

# CodeStructureExtractor 组件规范

## 1. 组件定位

`CodeStructureExtractor` 本地解析代码文件，提取确定性结构节点和边，不调用 LLM。它借鉴 Graphify 的 `extract.py`，但在 Legion 中应按语言族拆分实现，避免单文件膨胀。

## 2. 接口定义

```go
type CodeStructureExtractor interface {
    Extract(ctx context.Context, files []Path, opts ExtractOptions) (ExtractionFragment, error)
}
```

## 3. 输出节点和边

| 节点 | 边 |
|------|----|
| file/module | contains |
| class/type/struct/interface | inherits / implements / contains |
| function/method | calls / method |
| import/module | imports / imports_from |
| SQL table/view/column | foreign_key / joins / references |
| rationale comment | explains / why |

所有 AST 明确提取的边标记为 `EXTRACTED`。

## 4. 跨文件推断

提取分两阶段：

1. 单文件 AST 提取，记录 unresolved calls/imports。
2. 全量节点合并后解析跨文件关系。

推断规则：

- 有显式 import 证据时，跨文件 calls 可标记为 `EXTRACTED`。
- 同名多候选时跳过，避免误连。
- 外部库/stdlib dangling edge 不进入图谱。

## 5. 实现建议

| Graphify 做法 | Legion 建议 |
|---------------|-------------|
| 单个 `extract.py` 覆盖所有语言 | 按语言族拆为 `python_extractor`、`js_extractor`、`go_extractor` 等 |
| ProcessPoolExecutor | Go/Rust 实现可用 worker pool |
| AST cache | 写入 `knowledge_extraction_cache`，记录 extractor version |

