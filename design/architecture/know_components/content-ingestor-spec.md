---
id: "spec-know-content-ingestor-011"
title: "ContentIngestor 组件规范"
aliases: ["ContentIngestor规范", "内容接入器", "外部内容接入"]
type: "spec"
category: "design/architecture/know_components"
tags: ["component-spec", "llm-wiki", "ingest", "url", "office", "media"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-know-component-registry-000"

component_id: "K11"
layer: "K1"
depends_on:
  - "X03"
  - "X04"
  - "X05"
optional_deps: []
conflicts_with: []
required_by:
  - "K13"
assembly_profiles:
  - enterprise
---

# ContentIngestor 组件规范

## 1. 组件定位

`ContentIngestor` 负责把外部或非文本内容转换为 Wiki 可加工的 Markdown sidecar。它对应 Graphify 的 `ingest.py`、`detect.py` Office 转换、Google Workspace 转换和 `transcribe.py`。

## 2. 接口定义

```go
type ContentIngestor interface {
    IngestURL(ctx context.Context, req URLIngestRequest) (IngestedDocument, error)
    ConvertFile(ctx context.Context, path Path, opts ConvertOptions) (IngestedDocument, error)
    TranscribeMedia(ctx context.Context, path Path, opts TranscribeOptions) (IngestedDocument, error)
}
```

## 3. 接入类型

| 类型 | 转换结果 |
|------|----------|
| URL / arXiv | Markdown + source_url front matter |
| HTML | Markdown 正文 |
| `.docx` | Markdown 段落、标题、表格 |
| `.xlsx` | Sheet / table Markdown |
| `.gdoc/.gsheet/.gslides` | 通过已授权 workspace CLI 导出为 Markdown |
| video/audio | transcript Markdown |

## 4. 安全契约

- URL 获取必须通过 `SafeFetcher`。
- sidecar 路径必须通过 `PathGuard` 限定在 `know-out/converted/`。
- front matter 字段必须通过 `OutputSanitizer.YAMLString`。
- 原始二进制不写入知识块，只保留 source metadata。

## 5. 输出契约

`IngestedDocument` 必须包含：

| 字段 | 说明 |
|------|------|
| `doc_id` | 内容 hash 或外部 URL hash |
| `sidecar_path` | 转换后 Markdown 路径 |
| `source_uri` | 原始文件或 URL |
| `source_type` | url/office/google/media |
| `captured_at` | 采集时间 |
| `author/contributor` | 可选溯源 |

