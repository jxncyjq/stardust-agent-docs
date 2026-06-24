---
id: "spec-common-safe-fetcher-X03"
title: "SafeFetcher 组件规范"
aliases: ["SafeFetcher规范", "安全抓取器", "SSRF防护"]
type: "spec"
category: "design/architecture/common_components"
tags: ["component-spec", "common", "security", "ssrf", "fetch"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "index-common-components"

component_id: "X03"
layer: "X"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "K11"
assembly_profiles:
  - enterprise
---

# SafeFetcher 组件规范

## 1. 组件定位

`SafeFetcher` 是外部 URL 内容接入的公共安全组件，借鉴 Graphify `security.py` 的 URL 验证与 SSRF 防护。

## 2. 安全规则

- 仅允许 `http` / `https`。
- 阻断 `file://`、`ftp://`、`data:`。
- DNS 解析后拒绝 private、reserved、loopback、link-local、CGN 地址。
- 阻断云 metadata 主机名。
- 每次 redirect 重新校验 URL。
- fetch 时防 DNS rebinding。
- 二进制默认 50MB 上限，文本默认 10MB 上限。

## 3. 接口定义

```go
type SafeFetcher interface {
    Fetch(ctx context.Context, url string, opts FetchOptions) ([]byte, error)
    FetchText(ctx context.Context, url string, opts FetchOptions) (string, error)
    ValidateURL(ctx context.Context, url string) error
}
```

