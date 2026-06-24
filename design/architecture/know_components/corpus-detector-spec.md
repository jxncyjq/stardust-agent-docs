---
id: "spec-know-corpus-detector-010"
title: "CorpusDetector 组件规范"
aliases: ["CorpusDetector规范", "语料检测器", "文件发现"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "corpus", "detect", "graphify"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K10"
layer: "K1"
depends_on:
  - "X04"
optional_deps:
  - "X03"
conflicts_with: []
required_by:
  - "K12"
  - "K13"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

# CorpusDetector 组件规范

## 1. 组件定位

`CorpusDetector` 负责发现待进入 LLM Wiki 的语料，并分类为 code / document / paper / image / video / external shortcut。它借鉴 Graphify 的 `detect.py`：优先本地扫描，跳过敏感文件，生成增量 manifest。

## 2. 接口定义

```go
type CorpusDetector interface {
    Detect(ctx context.Context, root Path, opts DetectOptions) (DetectionResult, error)
    DetectIncremental(ctx context.Context, root Path, manifest Manifest) (IncrementalDetectionResult, error)
}

type DetectionResult struct {
    Files      map[CorpusType][]Path
    TotalFiles int
    TotalWords int
    Warnings   []string
}
```

## 3. 文件分类

| 类型 | 示例 |
|------|------|
| code | `.go`, `.rs`, `.py`, `.ts`, `.java`, `.sql` |
| document | `.md`, `.txt`, `.rst`, `.yaml`, `.docx`, `.xlsx` |
| paper | `.pdf` 或带 arXiv/DOI/abstract/citation 信号的文档 |
| image | `.png`, `.jpg`, `.webp`, `.svg` |
| video | `.mp4`, `.mov`, `.mp3`, `.wav` |
| google_workspace | `.gdoc`, `.gsheet`, `.gslides` |

## 4. 安全规则

- 通过 `PathGuard` 规范化根路径，禁止扫描工作区外部路径。
- 默认跳过 `.env`、密钥、证书、credential、token、private_key。
- 支持 `.knowignore` 与 `.knowinclude`，语法兼容 `.gitignore`。
- follow symlink 默认关闭；开启时必须检测循环。

## 5. 增量规则

`DetectIncremental` 使用内容 hash，而不是 mtime，生成：

| 字段 | 说明 |
|------|------|
| `new_files` | 新增或内容变化 |
| `unchanged_files` | 未变化，可复用缓存 |
| `deleted_files` | 已删除，需要从图谱和索引中 pruning |

## 6. 与 Graphify 的差异

Graphify 的 detector 输出到 `graphify-out/manifest.json`；Legion 应写入 `knowledge_manifests` 表，并加 `company_id`、`repo_id`、`snapshot_id`，支持多租户和多项目。

