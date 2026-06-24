---
id: "agent-release-001"
title: "Legion Agent 发布构建"
type: "guide"
category: "backend/agent"
tags: ["agent", "release", "build"]
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

# Legion Agent 发布构建

Legion Agent 通过 `scripts/release.ps1` 生成多平台二进制产物，并使用 Go `-ldflags` 注入版本信息。

## Version

查看当前二进制版本：

```powershell
go run ./cmd -- version --plain
```

输出示例：

```text
version=dev commit=unknown build_time=unknown
```

发布构建后：

```text
version=0.1.0 commit=<git-sha> build_time=2026-05-15T04:46:43Z
```

## Release Build

本地构建：

```powershell
.\scripts\release.ps1 -Version 0.1.0 -Commit <git-sha> -OutDir .\dist
```

默认生成：

| 产物 | 目标平台 |
|------|----------|
| `legion-agent-windows-amd64.exe` | Windows amd64 |
| `legion-agent-linux-amd64` | Linux amd64 |
| `legion-agent-linux-arm64` | Linux arm64 |

脚本会使用：

```text
go build -p=1 -buildvcs=false -ldflags ...
```

`-buildvcs=false` 用于避免本地或 CI 中 Git ownership/VCS 状态影响构建；`-p=1` 用于降低交叉编译并发，让构建在受限环境中更稳定。

## CI

GitHub Actions 在 test、vet、build、smoke 之后执行 release build：

```powershell
scripts/release.ps1 -Version 0.1.0-ci -Commit ${{ github.sha }} -OutDir dist
```

当前 CI 只验证产物可构建，不自动发布到包管理平台。

## Rollback

回滚时选择上一版对应平台产物，替换当前二进制，并确认：

```powershell
.\legion-agent-windows-amd64.exe version --plain
```

如使用 SQLite 持久化，回滚前先参考 [storage-ops.md](./storage-ops.md) 执行备份。
