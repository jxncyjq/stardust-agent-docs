---
id: "spec-agent-skill-installer-032"
title: "SkillInstaller 组件规范"
aliases: ["SkillInstaller规范", "技能安装器", "skill-installer-spec"]
type: "spec"
category: "design/architecture/agent_components"
tags: ["component-spec", "agent-engine", "skill", "installer"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-agent-component-registry-000"

component_id: "A32"
layer: "L3"
depends_on:
  - "A31"
optional_deps: []
conflicts_with: []
required_by:
  - "A30"
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# SkillInstaller 组件规范

## 1. 组件定位

`SkillInstaller` 从受信注册表或本地目录安装技能，负责下载、完整性校验、安全扫描、磁盘与 DB 元数据同步。

<!-- @end-section -->

<!-- @section: flow -->
---

## 2. 安装流程

1. 解析来源（registry / git / local）。
2. 下载或读取技能包。
3. 校验 SHA-256 与 manifest。
4. 调用 `SkillSecurityScanner`。
5. Critical 阻断，Warning 进入待审核。
6. 写入磁盘目录与技能元数据表。
7. 触发 `SkillSystem` 重载。

<!-- @end-section -->

<!-- @section: contracts -->
---

## 3. 行为契约

- 同名技能按 `name@version` 存储，禁止静默覆盖。
- disk wins：磁盘与 DB 冲突时以磁盘 manifest 为准并修复 DB。
- 安装记录必须包含来源 URL、hash、扫描报告 ID。

<!-- @end-section -->
