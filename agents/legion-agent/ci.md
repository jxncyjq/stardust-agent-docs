---
id: "agent-ci-001"
title: "Legion Agent CI 流水线"
type: "guide"
category: "backend/agent"
tags: ["agent", "ci", "pipeline"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "agent-index-001"
    relation: "related_to"
    path: "./index.md"
---

# Legion Agent CI 流水线

Legion Agent 的独立仓库 CI 位于 `legion/legionAgent/.github/workflows/agent-ci.yml`，用于在合并或发布前重复执行基础质量门禁。

## 触发条件

| 触发 | 范围 |
|------|------|
| push | `main`、`master` 分支中 Go 代码、依赖、Makefile、脚本或 workflow 变更 |
| pull_request | Go 代码、依赖、Makefile、脚本或 workflow 变更 |
| workflow_dispatch | 手动触发 |

## 验证步骤

| 步骤 | 命令 | 目的 |
|------|------|------|
| Test | `go test ./...` | 运行完整单元与集成测试 |
| Vet | `go vet ./...` | 执行 Go 标准静态检查 |
| Build | `go build ./cmd` | 验证服务入口可构建 |
| Smoke | `scripts/smoke.ps1` | 跑 demo、prompt、workflow、storage smoke 验收 |
| Compatibility | `go test ./internal/compat -count=1` | 验证最小配置、HTTP API 核心字段、Workflow DSL golden 契约 |
| Release build | `scripts/release.ps1 -Version 0.1.0-ci -Commit ${{ github.sha }} -OutDir dist` | 验证三平台发布产物可构建 |

CI 通过 `actions/setup-go` 读取 `go.mod` 中的 Go 版本，并使用 `go.sum` 作为依赖缓存键。

Compatibility 失败通常表示配置文件、HTTP 响应字段或 Workflow DSL 的最小契约发生破坏；需要显式调整版本策略并补迁移说明。
