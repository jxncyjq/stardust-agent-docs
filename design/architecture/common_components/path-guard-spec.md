---
id: "spec-common-path-guard-X04"
title: "PathGuard 组件规范"
aliases: ["PathGuard规范", "路径守卫", "路径沙盒"]
type: "spec"
category: "design/architecture/common_components"
tags: ["component-spec", "common", "security", "path", "sandbox"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "index-common-components"

component_id: "X04"
layer: "X"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "K10"
  - "K11"
  - "K12"
  - "K61"
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

# PathGuard 组件规范

## 1. 组件定位

`PathGuard` 统一处理路径规范化、目录边界校验和输出目录沙盒，防止路径穿越和任意文件读取。

## 2. 接口定义

```go
type PathGuard interface {
    ResolveInside(ctx context.Context, base Path, candidate Path) (Path, error)
    EnsureOutputPath(ctx context.Context, base Path, output Path) (Path, error)
    IsSensitive(path Path) bool
}
```

## 3. 行为契约

- 所有 path 先 resolve 再比较。
- `..`、symlink 逃逸、绝对路径逃逸必须拒绝。
- 支持敏感文件模式：`.env`、证书、密钥、token。
- 输出路径默认限制在工作区或指定 artifact 目录内。

