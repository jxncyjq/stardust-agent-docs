---
id: "plans-agent-p24-role-scoped-skills-001"
title: "P24 Role Scoped Skills 计划"
type: "implementation-plan"
category: "plans/agent"
tags: ["agent", "skills", "multi-agent", "role-scoped"]
version: "0.1.0"
created: "2026-06-01"
updated: "2026-06-01"
status: "active"
related_docs:
  - path: "./task-breakdown.md"
    relation: "tracked_by"
  - path: "../../agents/reference/reference-legion-agent-config-context-001.md"
    relation: "updates"
  - path: "../../agents/reference/reference-legion-agent-multi-agent-usage-001.md"
    relation: "updates"
---

# P24 Role Scoped Skills 计划

## 背景

P20 已经让多 Agent 按 `agent_id` 路由到不同 role、MaaS profile、context files 和 workspace。当前配置层也允许子 Agent 声明 `skills.install_root`，但运行时还没有把角色级 SkillSystem 接入 CognitiveCore，TUI/CLI 的 skill 管理也默认只操作根配置 `skills.install_root`。

## 目标

- 主 Agent 使用根配置 `skills.install_root`。
- 子 Agent 优先使用自己的 `skills.install_root`。
- 子 Agent 未配置 skills root 时继承根配置。
- CLI/TUI skill 管理可以显式选择 agent，避免把 writer/researcher 的技能安装到全局目录。

## 实施任务

| 任务 | 内容 | 验收 |
|------|------|------|
| AG-P24-001 | AgentRuntimeResolver 创建角色级 `skill.System` 并注入 CognitiveCore | 子 Agent prompt 包含角色目录中匹配到的 skill，不包含其他角色目录 skill |
| AG-P24-002 | `agent skill sync` 和 TUI `/skill` 支持 `--agent <name>` | 指定 writer 时使用 writer 配置中的 `skills.install_root` |
| AG-P24-003 | 更新配置示例、reference 和测试 | 文档说明继承规则和使用方式；测试通过 |

## 设计约束

- 不引入远端跨进程 skill 分发。
- 不改变现有全局 skills 行为。
- 不让子 Agent 默认获得写文件能力。
- 角色 skills 仅影响 prompt 中 `Mounted skills`，不改变 tool registry。
