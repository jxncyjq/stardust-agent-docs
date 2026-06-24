---
id: "analysis-evolver-index"
title: "Evolver 分析索引"
aliases: ["evolver index", "Evolver分析索引"]
type: "analysis"
category: "design/analysis/evolver"
tags: ["evolver", "gep", "index", "analysis"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-04"
author: "jxncyjq"
status: "review"
parent: null
children:
  - "analysis-evolver-overview-001"
  - "analysis-evolver-gep-002"
  - "analysis-evolver-atp-003"
  - "analysis-evolver-adapters-004"
  - "analysis-evolver-datamodels-005"
  - "analysis-evolver-insights-006"
  - "analysis-evolver-vs-hermes-007"
---

<!-- @section: index -->
# Evolver 分析文档索引

## 概述

本目录包含对 **Evolver**（EvoMap 开发的基于 GEP 的 AI Agent 自我进化引擎）的完整架构分析文档。Evolver 将临时性的 Prompt 调整转化为可审计、可复用的进化资产，是 Legion 项目在 Agent 自我改进和知识管理方面的重要参考。

- 分析对象：`evolver/`（JavaScript, Node.js >= 18, npm 包 `@evomap/evolver` v1.69.20）
- 文档数量：7 份分析文档 + 1 份索引
- 分析深度：协议级（GEP 进化协议、ATP 交易协议、信号引擎、资产系统）

## 文档列表

| 序号 | 文档 | 内容 | 状态 |
|------|------|------|------|
| 01 | [[01-overview|项目架构总览]] | 系统定位、策略预设、3 层信号、核心数据流 | review |
| 02 | [[02-gep-protocol|GEP 基因组进化协议分析]] | Gene/Capsule/Event 三元资产、3 层信号引擎、进化生命周期 | review |
| 03 | [[03-atp-protocol|ATP Agent 交易协议分析]] | Agent 间交易市场、商户/消费者/自动买家、Hub API | review |
| 04 | [[04-adapters-integration|适配器、CLI 与集成分析]] | 3 平台适配器、CLI 命令、本地代理邮箱、验证器子系统 | review |
| 05 | [[05-data-models|数据模型与资产系统分析]] | JSONL 存储、内容寻址、配置系统、安全机制 | review |
| 06 | [[06-evolver-insights|Evolver 洞察与 Legion 参考]] | 设计模式提炼、差异化方向、设计建议 | review |
| 07 | [[07-evolver-vs-hermes|Evolver vs Hermes 深度对比]] | 两项目深度对比、Evolver 独特优势、对 Legion 的启示 | review |

## 文档依赖关系

```
01-overview (总览)
  ├── 02-gep-protocol (进化协议)
  │     ├── 03-atp-protocol (交易协议)
  │     └── 05-data-models (数据模型)
  ├── 04-adapters-integration (集成方式)
  ├── 06-evolver-insights (Legion 参考) ← 汇总全部
  └── 07-evolver-vs-hermes (Hermes 对比) ← 横向对比
```

## 阅读路径

### 路径 1: 进化协议理解
`01-overview → 02-gep-protocol → 05-data-models`

适合快速理解 Evolver 的自我进化机制（信号→基因→固化）。

### 路径 2: 集成与部署
`01-overview → 04-adapters-integration → 03-atp-protocol`

适合了解如何将 Evolver 集成到 AI Agent 工作流中。

### 路径 3: Legion 设计参考
`01-overview → 06-evolver-insights → 02-gep-protocol`

适合从 Evolver 中提炼可复用的 Agent 自我改进方案。

### 路径 4: 横向对比分析
`01-overview → 07-evolver-vs-hermes → [[../hermes/07-hermes-vs-evolver|Hermes 对比]]`

适合理解 Evolver 与 Hermes 的差异定位，为 Legion 设计决策提供参考。

## 关键词索引

| 关键词 | 相关文档 |
|--------|----------|
| GEP 协议 | [[02-gep-protocol|02]] |
| Gene (基因) | [[02-gep-protocol|02]], [[05-data-models|05]] |
| Capsule (胶囊) | [[02-gep-protocol|02]], [[05-data-models|05]] |
| EvolutionEvent | [[02-gep-protocol|02]], [[05-data-models|05]] |
| 3 层信号提取 | [[02-gep-protocol|02]] |
| 策略预设 | [[01-overview|01]], [[02-gep-protocol|02]] |
| 固化 (Solidify) | [[02-gep-protocol|02]] |
| ATP 交易协议 | [[03-atp-protocol|03]] |
| 商户/消费者代理 | [[03-atp-protocol|03]] |
| 自动买家 | [[03-atp-protocol|03]] |
| 代理邮箱 (Proxy) | [[04-adapters-integration|04]] |
| 平台适配器 | [[04-adapters-integration|04]] |
| 钩子脚本 | [[04-adapters-integration|04]] |
| 验证器沙箱 | [[04-adapters-integration|04]] |
| 内容寻址 (SHA-256) | [[05-data-models|05]] |
| JSONL 存储 | [[05-data-models|05]] |
| 泄漏扫描 | [[05-data-models|05]] |
| 技能蒸馏 | [[02-gep-protocol|02]] |
| 金丝雀检查 | [[05-data-models|05]], [[02-gep-protocol|02]] |
| Hermes 对比 | [[07-evolver-vs-hermes|07]] |

## 技术栈速览

| 层级 | 技术 |
|------|------|
| 语言 | JavaScript (Node.js >= 18) |
| 运行时依赖 | dotenv (唯一) |
| 构建依赖 | javascript-obfuscator |
| 数据存储 | JSON + JSONL 文件系统 |
| 网络协议 | HTTP/JSON (A2A Protocol) |
| 本地代理 | 自研 HTTP Server (端口 19820) |
| 平台集成 | Node.js 钩子脚本 |
| 安全 | AES-256-GCM, SHA-256 |
| Git | 必须（回滚/提交/爆炸半径依赖） |

## 四项目对比总览

| 维度 | claw-code | new-api | hermes-agent | evolver |
|------|-----------|---------|-------------|---------|
| 语言 | Rust | Go | Python | JavaScript |
| 定位 | Claude Code 替代 | LLM API 网关 | AI Agent 平台 | Agent 自我进化 |
| 参考价值 | Agent 运行时基础 | MaaS 模型调度 | Agent 平台生态 | 进化与知识管理 |
| 核心特色 | MCP/权限/压缩 | 40+渠道适配 | 200+技能市场 | 信号驱动进化 |
| 分析文档 | claude/ (13 份) | maas/ (8 份) | hermes/ (8 份) | evolver/ (8 份) |
| 代码可读性 | 完全开源 | 完全开源 | 完全开源 | 核心混淆 |

## 统计

- 分析覆盖 JavaScript 文件：97（`src/**/*.js` 实测）
- 核心模块：57 (GEP，含 `gep/validator/` 4 个) + 8 (ATP) + 9 (Ops) + 4 (Adapters，含 hookAdapter 基础设施)
- GEP 可读模块：26 个（约 236KB）
- GEP 混淆模块：27 个（约 2.9MB）
- 信号类型：20+ 种
- 策略预设：6 种（`balanced` / `innovate` / `harden` / `repair-only` / `early-stabilize` / `steady-state`，由 `src/gep/strategy.js::getStrategyNames()` 返回）
- 分析文档总字数：约 20000 字
- 对比文档：1 份 (Evolver vs Hermes)

<!-- @end-section -->

<!-- @section: related -->
## 相关目录

- [[../claude/index|Claw Code 分析索引]] — Agent 运行时参考项目（Rust）
- [[../maas/index|MaaS 模型调度层分析索引]] — 模型网关参考项目（Go）
- [[../hermes/index|Hermes Agent 分析索引]] — Agent 平台参考项目（Python）

<!-- @end-section -->
