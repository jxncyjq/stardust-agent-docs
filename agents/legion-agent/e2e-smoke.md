---
id: "agent-e2e-smoke-001"
title: "Legion Agent 端到端 Smoke 验收"
type: "guide"
category: "backend/agent"
tags: ["agent", "e2e", "smoke", "test"]
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

# Legion Agent 端到端 Smoke 验收

本文档记录 Legion Agent 当前可重复执行的端到端验证入口。命令均在仓库目录 `legion/legionAgent` 下执行。

## 一键 Smoke

```powershell
.\scripts\smoke.ps1
```

`.\scripts\smoke.ps1` 会依次执行：

| 目标 | 作用 |
|------|------|
| `demo-smoke` | 运行 `agent run --demo --plain`，验证 demo 闭环 |
| `prompt-smoke` | 运行 `agent run --plain --prompt ...`，验证自定义 prompt 路径 |
| `workflow-smoke` | 运行 workflow subworkflow 聚焦测试 |
| `storage-smoke` | 运行 SQLite 跨进程恢复聚焦测试 |

## 单项命令

```powershell
make smoke
make demo-smoke
make prompt-smoke
make workflow-smoke
make storage-smoke
```

Windows 环境如果没有安装 `make`，直接使用 `.\scripts\smoke.ps1` 即可。

## 完整工程验证

```powershell
make test
make vet
make build
```

`make test` 对应 `go test ./...`，`make vet` 对应 `go vet ./...`，`make build` 对应 `go build ./cmd`。
