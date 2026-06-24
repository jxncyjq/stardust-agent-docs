---
id: "plans-agent-skill-system-001"
title: "Agent 技能系统计划"
type: "plan"
category: "plans/agent"
tags: ["plan", "agent", "skills", "security"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
related_docs:
  - path: "../../design/architecture/agent_components/skill-system-spec.md"
    relation: "implements"
  - path: "../../design/architecture/agent_components/skill-security-scanner-spec.md"
    relation: "implements"
  - path: "../../design/architecture/agent_components/skill-installer-spec.md"
    relation: "implements"
---

# Agent 技能系统计划

## 目标

在 enterprise 阶段实现 A30/A31/A32，让 Agent 可以从受控来源加载技能，并在每次任务执行前按任务上下文注入有限技能。

## 实施边界

| 组件 | Go 包 | 职责 |
|------|-------|------|
| A30 SkillSystem | `internal/skill` | 扫描、去重、排序、按任务注入技能 |
| A31 SkillSecurityScanner | `internal/skill` | 对技能内容做安全扫描和风险分级 |
| A32 SkillInstaller | `internal/skill` | 从注册表安装技能，校验完整性，维护磁盘/DB 同步 |

## 分期

### P5.1 只读技能加载

- 支持本地目录扫描。
- 支持技能元数据解析。
- 同一任务最多注入 3 个技能。
- A31 对本地技能执行注入、SSRF、路径穿越基础规则。

### P5.2 安装与同步

- A32 支持 registry manifest。
- 下载后校验 SHA-256。
- 磁盘与 repository 双向同步。
- SkillInstall 触发 A62-full 审批。

### P5.3 运行时注入

- A00 CognitiveCore 通过 A30 获取任务相关技能。
- A30 不直接调用 LLM，不直接执行工具。
- 被标记为 risky 或 frozen 的技能不得注入。

## 数据对象

| 对象 | 字段 |
|------|------|
| `Skill` | id、name、source、version、path、hash、risk_level、status |
| `SkillScanFinding` | skill_id、rule_id、severity、message、location |
| `SkillInjection` | task_id、skill_id、rank、reason、created_at |

## 必测场景

- 恶意技能包含路径穿越片段时被 A31 标记为 Critical。
- 同名同版本技能只保留一个候选。
- risky 技能不会被 CognitiveCore 注入。
- 安装 hash 不匹配时失败并写入审计事件。
