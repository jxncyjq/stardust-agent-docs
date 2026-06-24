---
id: "spec-agent-skill-security-scanner-031"
title: "SkillSecurityScanner 组件规范"
aliases: ["SkillSecurityScanner规范", "技能安全扫描器", "skill-security-scanner-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "skill", "security", "scanner"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A31"
layer: "L3"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "A30"
  - "A32"
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# SkillSecurityScanner 组件规范

## 1. 组件定位

`SkillSecurityScanner` 在技能安装、同步和运行时注入前执行安全扫描，防止 prompt injection、SSRF、路径穿越、秘密外泄和危险命令。

<!-- @end-section -->

<!-- @section: rules -->
---

## 2. 规则分级

| 等级 | 行为 | 示例 |
|------|------|------|
| Critical | 阻断安装/注入 | 要求泄露密钥、禁用安全策略、访问任意 URL |
| Warning | 允许但提示审核 | 高权限 shell、宽泛文件匹配 |
| Info | 记录 | 大文件、缺少示例 |

内置规则覆盖：prompt override、secret exfiltration、SSRF、path traversal、unsafe shell、wildcard delete、credential request、policy bypass、network beacon、binary drop、hidden instruction、unbounded recursion、license missing。

<!-- @end-section -->

<!-- @section: interface -->
---

## 3. 接口定义

```go
type SkillSecurityScanner interface {
    Scan(ctx context.Context, skill SkillPackage) (SkillScanReport, error)
}
```

<!-- @end-section -->
