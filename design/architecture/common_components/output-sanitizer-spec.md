---
id: "spec-common-output-sanitizer-X05"
title: "OutputSanitizer 组件规范"
aliases: ["OutputSanitizer规范", "输出净化器", "标签净化"]
type: "spec"
category: "design/architecture/common_components"
tags: ["component-spec", "common", "security", "html", "yaml", "mcp"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "index-common-components"

component_id: "X05"
layer: "X"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "K11"
  - "K13"
  - "K20"
  - "K60"
  - "K61"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

# OutputSanitizer 组件规范

## 1. 组件定位

`OutputSanitizer` 统一处理知识图谱导出、MCP 输出、Markdown 报告、HTML/YAML 片段中的标签、路径和文本净化。

## 2. 接口定义

```go
type OutputSanitizer interface {
    Label(text string) string
    HTML(text string) string
    YAMLString(text string) string
    MarkdownInline(text string) string
    Truncate(text string, maxLen int) string
}
```

## 3. 行为契约

- 去除 ASCII control chars。
- label 默认最大 256 字符。
- HTML 输出必须 escape。
- YAML 双引号字符串必须转义反斜杠、引号、换行、tab、NUL 和 C0 控制字符。
- MCP 文本输出不得包含 ANSI 控制序列。

