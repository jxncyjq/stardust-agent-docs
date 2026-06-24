---
id: "agent-skill-registry-001"
title: "Legion Agent Skill Registry"
type: "reference"
category: "backend/agent"
tags: ["agent", "skill", "registry"]
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

# Legion Agent Skill Registry

P12 为 A30-A32 增加远端 Skill registry 同步入口。同步器只负责读取 registry index、下载 manifest 和技能内容，真正安装仍复用既有 `SkillInstaller`，因此 hash 校验、A31 安全扫描、quarantine 和审计链保持一致。

## Registry Index

入口文件是 JSON：

```json
{
  "skills": [
    {
      "manifest_url": "https://registry.example.test/go-testing.json"
    }
  ]
}
```

当前支持 `http://` 与 `https://` URL。其他 scheme 会被拒绝。

## Manifest

每个 manifest 使用既有安装格式：

```json
{
  "id": "go-testing",
  "name": "Go Testing",
  "version": "1.0.0",
  "content_path": "https://registry.example.test/go-testing/SKILL.md",
  "sha256": "<skill-content-sha256>",
  "tags": ["go", "test"]
}
```

同步器会把远端内容下载为本地临时 manifest，再交给 installer。installer 会重新计算 `SKILL.md` 的 SHA-256；不匹配时不会写入安装目录。

## CLI

```powershell
agent skill sync --config agent.json
agent skill sync --registry-url https://registry.example.test/index.json --install-root .\skills
```

配置文件：

```json
{
  "skills": {
    "registry_url": "https://registry.example.test/index.json",
    "install_root": "skills"
  }
}
```

输出为稳定键值格式：

```text
skill_sync installed=1 quarantined=0 failed=0
```

## 安全行为

- Critical 扫描结果会进入 `quarantined`，不会安装到启用目录。
- Hash mismatch 会计入 `failed`。
- SQLite 存储配置下，skill metadata、scan findings 和 `skill_installed` / `skill_quarantined` 审计事件会写入同一仓储。
- 非 SQLite 模式使用内存仓储，仅保证本次命令安装文件落盘。

