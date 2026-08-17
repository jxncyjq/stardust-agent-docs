---
id: "analysis-dsh-index"
title: "DeepSeek Harness 分析索引"
aliases: ["dsh index", "DeepSeek Harness 分析索引", "deepseek-harness analysis"]
type: "analysis"
category: "design/analysis/deepseek-harness"
tags: ["deepseek-harness", "dsh", "index", "analysis", "cordis", "agent"]
version: "1.0.0"
created: "2026-08-15"
updated: "2026-08-15"
author: "jxncyjq"
status: "review"
parent: null
children:
  - "analysis-dsh-overview-001"
  - "analysis-dsh-architecture-002"
  - "analysis-dsh-plugin-system-003"
  - "analysis-dsh-capabilities-004"
  - "analysis-dsh-insights-005"
  - "analysis-dsh-wasm-porting-006"
related_docs:
  - id: "analysis-dsh-overview-001"
    relation: "related_to"
    path: "./01-overview.md"
  - id: "analysis-dsh-architecture-002"
    relation: "related_to"
    path: "./02-architecture.md"
  - id: "analysis-dsh-plugin-system-003"
    relation: "related_to"
    path: "./03-plugin-system.md"
  - id: "analysis-dsh-capabilities-004"
    relation: "related_to"
    path: "./04-core-capabilities.md"
  - id: "analysis-dsh-insights-005"
    relation: "related_to"
    path: "./05-insights.md"
  - id: "analysis-dsh-wasm-porting-006"
    relation: "related_to"
    path: "./06-wasm-plugin-porting.md"
---

# DeepSeek Harness 分析索引

<!-- @section: overview -->
## 概述

对 DeepSeek AI 开源 agent harness **DeepSeek Harness（`dsh`）** 的代码库分析：系统架构、插件系统、核心能力，以及对 Legion 的借鉴评估。

| 项 | 值 |
|---|---|
| 仓库 | `f:\source\github\deepseek-harness` |
| 上游 | https://github.com/deepseek-ai/deepseek-harness |
| 版本 | `0.1.0-rc.5`（开发者预览） |
| 分析快照 | HEAD `47f9438`，2026-08-13 |
| 协议 | MIT |
| 技术栈 | TypeScript / ESM / Node ≥22.19 / pnpm workspaces / vendored Cordis |
| 规模 | 219 个工作区包 + 2 个应用（cli / web） |

一句话概括：**一切皆插件**——模型适配器、工具注册表、会话日志、agent 主循环本身都是挂在 Cordis 上下文树上的插件，全部可由配置替换；append-only 的 session log 是唯一真相源，模型历史由它派生。

<!-- @end-section -->

<!-- @section: index -->
## 文档目录

| 文档 | 内容 |
|---|---|
| [[analysis-dsh-overview-001\|01 项目总览]] | 技术栈、仓库布局、包分组、运行入口、工程约束 |
| [[analysis-dsh-architecture-002\|02 系统架构]] | Cordis 微内核五概念、派发模式、profile/bundle 组合、核心服务图、事件三域、turn/step 生命周期、session log 与持久化 |
| [[analysis-dsh-plugin-system-003\|03 插件系统]] | 插件形态、capability seam 三角色、scope 与 shadowing、拦截点选型、agent preset、动态自修改、HMR、Typert 类型图 RPC、防腐护栏 |
| [[analysis-dsh-capabilities-004\|04 核心能力]] | 工具执行管线、工具清单、执行与隔离四层、委派与编排、上下文治理、知识与人格、人机协作、数据运维、对外互操作 |
| [[analysis-dsh-insights-005\|05 评估与借鉴]] | 六个真正强的设计、六项代价与风险、对 Legion 的三档可移植性评估 |
| [[analysis-dsh-wasm-porting-006\|06 WASM 插件移植与选型]] | dsh 插件模型向 Go+WASM 的分拣（可抄/必改/WASM 独有）、Go WASM 生态 2026 现状、Go 插件机制全景对比、agent 场景选型 |

<!-- @end-section -->

<!-- @section: quickref -->
## 快速检索

| 想知道 | 看哪 |
|---|---|
| 一个请求是怎么跑完的？ | [[analysis-dsh-architecture-002]] § Turn / Step 生命周期 |
| 新功能应该挂在哪？ | [[analysis-dsh-architecture-002]] § 新行为该挂在哪 |
| 什么是 capability seam？ | [[analysis-dsh-plugin-system-003]] § Capability Seam |
| 怎么做 per-agent 工具集？ | [[analysis-dsh-plugin-system-003]] § Scope |
| 工具执行有哪些拦截点？ | [[analysis-dsh-capabilities-004]] § 工具执行管线 |
| 沙箱边界到哪？ | [[analysis-dsh-capabilities-004]] § 执行与隔离 |
| 上下文爆了怎么办？ | [[analysis-dsh-capabilities-004]] § 上下文治理 |
| 我们该抄什么？ | [[analysis-dsh-insights-005]] § 对 Legion 的对照与可移植点 |
| WASM 插件该选什么底座？ | [[analysis-dsh-wasm-porting-006]] § Go 的 WASM 生态 / agent 场景选型 |
| waterfall 在 WASM 下怎么做？ | [[analysis-dsh-wasm-porting-006]] § 必须改造 |

<!-- @end-section -->

<!-- @section: upstream-docs -->
## 上游关键文档（原仓）

| 文件 | 内容 |
|---|---|
| `AGENTS.md` | 仓库规约、约定、防腐条款（根 `CLAUDE.md` 是它的软链） |
| `docs/architecture.md` | 架构主文档，改 `packages/` 前必读 |
| `docs/cordis-primer.md` | Cordis 五概念与 waterfall 语义 |
| `docs/glossary.md` | seam / scope / turn / step / round / Ralph 的规范定义 |
| `docs/capability-seams.md` | 生成的服务与 seam 全景图 |
| `docs/tool-execution-pipeline.md` | 生成的工具管线流程图 |
| `docs/agent-lifecycle.md` | 生成的 turn/step 时序图 |
| `docs/subsystems/*.md` | 每个子系统一页，含从源码生成的 Cordis API 段 |
| `docs/cookbook/extension-cookbook.md` | 「产品特性 → 插件机制」映射表 |
| `.agents/notes/` | 决策记录（Agent Notes），非平凡改动必须附带 |

<!-- @end-section -->

## 相关文档

- [[analysis-dsh-overview-001|01 项目总览]]
- [[analysis-dsh-architecture-002|02 系统架构]]
- [[analysis-dsh-plugin-system-003|03 插件系统]]
- [[analysis-dsh-capabilities-004|04 核心能力]]
- [[analysis-dsh-insights-005|05 评估与借鉴]]
- [[analysis-dsh-wasm-porting-006|06 WASM 插件移植与选型]]
- [[analysis-hermes-index|Hermes Agent 分析索引]] — 同类 harness 的对照分析
