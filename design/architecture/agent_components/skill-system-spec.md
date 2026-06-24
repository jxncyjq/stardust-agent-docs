---
id: "spec-agent-skill-system-030"
title: "SkillSystem 组件规范"
aliases: ["SkillSystem规范", "技能系统", "skill-system-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "skill", "context"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A30"
layer: "L3"
depends_on:
  - "A31"
optional_deps:
  - "A32"
conflicts_with: []
required_by:
  - "A00"
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# SkillSystem 组件规范

## 1. 组件定位

`SkillSystem` 负责发现、加载、筛选并向 `CognitiveCore` 注入任务相关技能。技能只改变上下文与可用工具，不直接执行任务。

<!-- @end-section -->

<!-- @section: sources -->
---

## 2. 技能来源

| 来源 | 说明 |
|------|------|
| built-in | 平台内置技能 |
| workspace | 仓库 `.agents/skills` |
| user | 用户级技能目录 |
| org | 组织共享技能 |
| registry | 远程注册表安装 |
| generated | GEP 生成但已审核技能 |
| plugin | 插件提供技能 |

<!-- @end-section -->

<!-- @section: injection -->
---

## 3. 注入规则

- 每个任务最多注入 3 个技能。
- Critical 扫描未通过的技能不得注入。
- 技能内容进入 P6，低于安全红线与任务指令。
- 技能选择必须记录匹配原因和版本。

<!-- @end-section -->
