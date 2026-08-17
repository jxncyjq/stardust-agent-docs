---
id: "analysis-pi-index"
title: "pi-agent 分析索引"
aliases: ["pi index", "pi-agent 分析索引", "pi mono 分析"]
type: "design"
category: "design/analysis/pi-agent"
tags: ["pi-agent", "index", "analysis"]
version: "1.0.0"
created: "2026-08-15"
updated: "2026-08-15"
author: "jxncyjq"
status: "review"
parent: null
children:
  - "analysis-pi-overview-001"
  - "analysis-pi-architecture-002"
  - "analysis-pi-harness-003"
  - "analysis-pi-tools-004"
  - "analysis-pi-skills-005"
  - "analysis-pi-extensions-006"
  - "analysis-pi-insights-007"
related_docs:
  - id: "analysis-hermes-index"
    relation: "related_to"
    path: "../hermes/index.md"
---

# pi-agent 分析文档索引

<!-- @section: index -->
## 概述

本目录是对 **pi**（`earendil-works/pi-mono`，npm `@earendil-works/pi-*`）的架构分析，**重点放在 harness / skills / mcp / tools 四条能力装载通路**。

- 分析对象：`F:\source\github\pi`
- 快照：commit `086c32e`（2026-08-15），CLI 版本 `0.84.2`
- 技术栈：TypeScript 5.9 / Node ≥22.19 / Bun 二进制 / npm workspaces
- 文档：7 份分析 + 1 份索引

一句话结论：**pi 内核最小、扩展点最大；它用 TypeScript 扩展 + Agent Skills 文件取代了 MCP，并正在用一份 2941 行规范把执行层重写成可崩溃恢复的状态机。**
<!-- @end-section -->

<!-- @section: docs -->
## 文档列表

| 序号 | 文档 | 内容 | 状态 |
|------|------|------|------|
| 01 | [[analysis-pi-overview-001\|项目架构总览]] | 定位、monorepo 包分层、四种运行模式、设计哲学、能力装载全景、两代 harness | review |
| 02 | [[analysis-pi-architecture-002\|运行时架构与主循环]] | 三层运行时、AgentMessage 双次变换、runLoop 精确语义、并行工具三段式、AgentSession、会话树、远程协议 | review |
| 03 | [[analysis-pi-harness-003\|Harness 分层与下一代规范]] | 现役 vs 规范版、三仓存储、lane、operation 状态机、工具持久化语义、11 个 hook、恢复模型、交付切片 | review |
| 04 | [[analysis-pi-tools-004\|工具系统与装载机制]] | AgentTool/ToolDefinition 双类型、7 个内置工具、`_refreshToolRegistry` 装配、addedToolNames、提示词贡献、拦截链、输出治理 | review |
| 05 | [[analysis-pi-skills-005\|技能系统]] | 渐进式披露、发现规则与去重、frontmatter 校验、`/skill:name` 展开、两份加载器对比、与 prompt 模板分工 | review |
| 06 | [[analysis-pi-extensions-006\|扩展系统与 No-MCP 立场]] | MCP 取证、jiti 三套模块解析、ExtensionAPI 全貌、28 个事件、runtime 陈旧检测、示例清单、安全边界 | review |
| 07 | [[analysis-pi-insights-007\|对 Legion 的启示]] | 8 条可提炼模式、可恢复执行模型对照表、No-MCP 的可移植与不可移植部分、不建议照搬项、候选动作项 | review |
<!-- @end-section -->

<!-- @section: reading-paths -->
## 阅读路径

### 路径 1：只关心「工具/技能/扩展怎么装进来」（本次分析重点）
`01-overview（能力装载全景一节）→ 04-tools → 05-skills → 06-extensions-mcp`

### 路径 2：关心执行引擎与可靠性
`01-overview → 02-architecture → 03-harness`

### 路径 3：只要结论
`07-insights`（含全部可行动建议）

## 文档依赖关系

```
01-overview (总览)
  ├── 02-architecture (运行时)
  │     └── 03-harness (harness 分层 + 下一代规范)
  ├── 04-tools (工具装载)  ←── 06 依赖
  ├── 05-skills (技能装载)
  ├── 06-extensions-mcp (扩展 + No-MCP)
  └── 07-insights (Legion 参考) ← 汇总全部
```
<!-- @end-section -->

<!-- @section: key-findings -->
## 关键结论速查

| 主题 | 结论 |
|------|------|
| **MCP** | **零实现**，且是明确的产品立场。全仓库 `mcp` 只出现在 OAuth scope 字符串、一条注释、README 的哲学声明里。替代方案 = Skills（CLI 工具 + README）+ Extensions（可自行接 MCP） |
| **Tools** | 内核 `AgentTool` / 产品 `ToolDefinition` 双类型；7 个内置工具（默认只激活 read/bash/edit/write）；`grep`/`find` 依赖外部 rg/fd 并自动下载；注册表刷新时「新出现的工具自动激活」实现扩展热注册 |
| **Skills** | Agent Skills 标准实现；只注入 name/description/location，全文由模型 `read`；`/skill:name` 可强制加载；项目级技能受 project trust 门控；可直接复用 `~/.claude/skills`、`~/.codex/skills` |
| **Extensions** | jiti 运行时加载 TS，三套模块解析策略（Bun virtualModules / TS 源码 / dist alias）；可注册工具/命令/快捷键/CLI flag/渲染器，可换 LLM provider（含 OAuth）；28 个事件、进程内全权限、无沙箱 |
| **Harness** | 现役 = `AgentSession`+`Agent`+`agent-loop`；下一代 `AgentHarness` 仍是脚手架，但已有 2941 行完整规范（三仓存储、lane、operation 状态机、`replay: never\|safe`、11 个 hook、崩溃恢复、16 个交付切片） |
<!-- @end-section -->

## 相关文档

- [[analysis-hermes-index|Hermes Agent 分析索引]] — 另一套 Agent 框架分析，工具/技能/插件章节可横向对比
- [[reference-docs-index-001|文档索引]]
