---
id: "index-analysis-000"
title: "Claw Code 分析文档索引"
aliases: ["分析索引", "analysis index"]
type: "reference"
category: "design/analysis/claude"
tags: ["index", "analysis", "claw-code"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "published"
parent: null
children:
  - "analysis-clawcode-overview-001"
  - "analysis-clawcode-rust-002"
  - "analysis-clawcode-python-003"
  - "analysis-clawcode-checklist-004"
  - "analysis-clawcode-flows-005"
  - "analysis-clawcode-datamodel-006"
  - "analysis-clawcode-gap-007"
  - "design-maas-scheduling-008"
  - "design-agent-runtime-009"
  - "design-tool-skill-010"
  - "design-permission-sandbox-011"
  - "design-session-memory-012"
  - "design-plugin-protocol-013"
---

<!-- @section: overview -->
# Claw Code 分析文档索引

本目录包含对 claw-code（Anthropic Claude Code 的 Rust 重写实现）的全面架构分析，以及基于分析结果产出的 Legion 项目技术设计文档。

共 **13 篇文档**，分为两大类：**分析文档**（01-07）和 **设计文档**（08-13）。
<!-- @end-section -->

---

<!-- @section: analysis-docs -->
## 一、Claw Code 分析文档（01-07）

对 claw-code 源码的深入分析，覆盖架构、模块、流程、数据模型和差距评估。

| 序号 | 文档 | 类型 | 说明 |
|------|------|------|------|
| 01 | [[01-overview\|项目架构总览]] | `analysis` | 整体架构、crate 关系、数据流、提供商支持 |
| 02 | [[02-rust-crates-analysis\|Rust Crate 功能模块分析]] | `analysis` | 9 个 crate 的模块清单和公共 API |
| 03 | [[03-python-subsystems-analysis\|Python 子系统功能分析]] | `analysis` | 36 个操作模块 + 29 个子系统占位包 |
| 04 | [[04-requirements-checklist\|系统化需求清单]] | `analysis` | 218 项需求、完成度统计（96.8%） |
| 05 | [[05-architecture-flows\|关键架构流程分析]] | `analysis` | 对话状态机、工具调度、权限检查、压缩流程 |
| 06 | [[06-data-models\|核心数据模型分析]] | `analysis` | 类型系统、序列化策略、数据转换关系 |
| 07 | [[07-gap-analysis\|差距分析与待完成工作]] | `analysis` | P0/P1/P2 技术债务、上游差距、路线图 |

<!-- @end-section -->

---

<!-- @section: design-docs -->
## 二、Legion 技术设计文档（08-13）

基于 claw-code 架构分析，为 Legion 项目产出的技术设计文档。

| 序号 | 文档 | 类型 | 对应 Legion 模块 | 参考 claw-code |
|------|------|------|------------------|---------------|
| 08 | [[08-MaaS模型调度层技术设计\|MaaS 模型调度层技术设计]] | `design` | 模型注册 + 智能路由 + 配额管控 | `api` crate（10 模块） |
| 09 | [[09-Agent运行时引擎技术设计\|Agent 运行时引擎技术设计]] | `design` | 认知内核 + 对话状态机 + 记忆蒸馏 | `runtime` crate（43 模块） |
| 10 | [[10-工具与技能系统设计\|工具与技能系统设计]] | `design` | 工具注册 + 技能市场 + 命令系统 | `tools` + `commands` crate |
| 11 | [[11-权限与安全模型设计\|权限与安全模型设计]] | `design` | 四层权限 + 沙箱 + Bash 校验 | `permissions` + `sandbox` |
| 12 | [[12-会话与记忆管理设计\|会话与记忆管理设计]] | `design` | 会话持久化 + 压缩 + 记忆转化 | `session` + `compact` |
| 13 | [[13-插件与协议扩展设计\|插件与协议扩展设计]] | `design` | 插件系统 + MCP + Adapter + OAuth | `plugins` + `mcp_*` |

<!-- @end-section -->

---

<!-- @section: dependency-graph -->
## 三、文档依赖关系

分析文档（01-07）的阅读顺序：

```
01-overview ──┬── 02-rust-crates-analysis ──┬── 05-architecture-flows
              │                              ├── 06-data-models
              ├── 03-python-subsystems       └── 04-requirements-checklist
              └── 07-gap-analysis                └── 07-gap-analysis
```

设计文档（08-13）的技术栈依赖链：

```
08-MaaS模型调度层
  └── 09-Agent运行时引擎
        └── 10-工具与技能系统
              └── 11-权限与安全模型
                    └── 12-会话与记忆管理
                          └── 13-插件与协议扩展
```

设计文档与分析文档的参考关系：

```
08-MaaS ────────── references ──→ 02-rust-crates, 06-data-models
09-Agent ───────── references ──→ 02-rust-crates, 05-flows
10-工具技能 ────── references ──→ 02-rust-crates
11-权限安全 ────── references ──→ 05-flows
12-会话记忆 ────── references ──→ 05-flows, 06-data-models
13-插件协议 ────── references ──→ 02-rust-crates
```

<!-- @end-section -->

---

<!-- @section: keywords -->
## 四、关键词索引

| 关键词 | 相关文档 |
|--------|----------|
| MaaS / 模型调度 | [[08-MaaS模型调度层技术设计]], [[04-requirements-checklist]] |
| Agent / 运行时 | [[09-Agent运行时引擎技术设计]], [[05-architecture-flows]] |
| 工具系统 / 技能 | [[10-工具与技能系统设计]], [[02-rust-crates-analysis]] |
| 权限 / 安全 / 沙箱 | [[11-权限与安全模型设计]], [[05-architecture-flows]] |
| 会话 / 记忆 / 压缩 | [[12-会话与记忆管理设计]], [[06-data-models]] |
| 插件 / MCP / OAuth | [[13-插件与协议扩展设计]], [[02-rust-crates-analysis]] |
| 数据模型 / 类型 | [[06-data-models]] |
| API 客户端 / 提供商 | [[08-MaaS模型调度层技术设计]], [[02-rust-crates-analysis]] |
| 对话状态机 / SSE | [[09-Agent运行时引擎技术设计]], [[05-architecture-flows]] |
| 差距分析 / 技术债务 | [[07-gap-analysis]], [[04-requirements-checklist]] |
| Python 子系统 / 移植 | [[03-python-subsystems-analysis]] |
| 项目概览 / 架构 | [[01-overview]] |

<!-- @end-section -->
