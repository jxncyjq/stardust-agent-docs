---
id: "analysis-maas-index"
title: "MaaS 模型调度层分析索引"
aliases: ["maas index", "MaaS分析索引", "new-api analysis index"]
type: "analysis"
category: "design/analysis/maas"
tags: ["maas", "new-api", "index", "analysis"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: null
children:
  - "analysis-newapi-overview-001"
  - "analysis-newapi-modules-002"
  - "analysis-newapi-channel-003"
  - "analysis-newapi-billing-004"
  - "analysis-newapi-middleware-005"
  - "analysis-newapi-datamodels-006"
  - "analysis-newapi-insights-007"
---

<!-- @section: index -->
# MaaS 模型调度层 — New API 分析文档索引

## 概述

本目录包含对 **New API**（下一代 LLM 网关与 AI 资产管理平台）的完整架构分析文档。New API 聚合数十家上游 AI 提供商（`constant/channel.go` 中分配 48 个 `ChannelType*` 常量，由 37 个 `relay/channel/<name>/` 适配器实现，OpenAI 兼容渠道共享 `openai/` 适配器），对外提供统一的 OpenAI 兼容 API，是 Legion MaaS 模型调度层的重要参考项目。

- 分析对象：`new-api/`（Go 1.22+, Gin, GORM v2）
- 文档数量：7 份分析文档 + 1 份索引
- 分析深度：模块级（每个 Go 包的职责、接口、数据流）

## 文档列表

| 序号 | 文档 | 内容 | 状态 |
|------|------|------|------|
| 01 | [[01-overview|项目架构总览]] | 系统定位、技术栈、分层架构、核心数据流 | review |
| 02 | [[02-module-analysis|Go 模块功能分析]] | 15 个 Go 包的职责、依赖关系、设计模式 | review |
| 03 | [[03-channel-adapter-system|渠道适配器系统分析]] | 37 个适配器的接口设计、委托模式、流处理 | review |
| 04 | [[04-billing-quota-system|计费与配额系统分析]] | 三种定价模式、BillingSession 生命周期、订阅系统 | review |
| 05 | [[05-middleware-and-flow|中间件与请求流程分析]] | 21 个中间件的完整管道、Context 数据流、限流系统 | review |
| 06 | [[06-data-models|核心数据模型分析]] | 25 张表结构、实体关系、三层缓存策略 | review |
| 07 | [[07-maas-insights|MaaS 洞察与 Legion 参考]] | 设计模式提炼、差异化方向、设计建议 | review |

## 文档依赖关系

```
01-overview (总览)
  ├── 02-module-analysis (模块分析)
  │     ├── 03-channel-adapter-system (渠道适配器)
  │     └── 04-billing-quota-system (计费配额)
  ├── 05-middleware-and-flow (中间件流程)
  ├── 06-data-models (数据模型)
  └── 07-maas-insights (Legion 参考) ← 汇总全部
```

## 阅读路径

### 路径 1: 架构理解
`01-overview → 02-module-analysis → 05-middleware-and-flow → 03-channel-adapter-system`

适合快速理解 New API 的整体架构和核心流程。

### 路径 2: 计费系统
`01-overview → 04-billing-quota-system → 06-data-models`

适合深入了解计费、定价、订阅系统的设计。

### 路径 3: Legion 设计参考
`01-overview → 07-maas-insights → 03-channel-adapter-system → 04-billing-quota-system`

适合从 New API 中提炼可复用的设计模式和需要注意的坑点。

## 关键词索引

| 关键词 | 相关文档 |
|--------|----------|
| Adaptor 接口 | [[03-channel-adapter-system|03]] |
| BillingSession | [[04-billing-quota-system|04]] |
| 阶梯表达式计费 | [[04-billing-quota-system|04]] |
| Context 数据传递 | [[05-middleware-and-flow|05]] |
| 委托模式 | [[03-channel-adapter-system|03]] |
| 跨分组重试 | [[05-middleware-and-flow|05]], [[03-channel-adapter-system|03]] |
| 幂等预扣费 | [[04-billing-quota-system|04]], [[06-data-models|06]] |
| 三数据库兼容 | [[06-data-models|06]] |
| SSE 流处理 | [[03-channel-adapter-system|03]] |
| 订阅系统 | [[04-billing-quota-system|04]], [[06-data-models|06]] |
| Token Auth | [[05-middleware-and-flow|05]] |
| 信任额度旁路 | [[04-billing-quota-system|04]] |
| BodyStorage | [[05-middleware-and-flow|05]] |
| Channel Cache | [[06-data-models|06]], [[03-channel-adapter-system|03]] |
| 多 Key 管理 | [[03-channel-adapter-system|03]], [[06-data-models|06]] |

## 技术栈速览

| 层级 | 技术 |
|------|------|
| 语言 | Go 1.22+ |
| HTTP 框架 | Gin |
| ORM | GORM v2 |
| 数据库 | SQLite / MySQL 5.7.8+ / PostgreSQL 9.6+ |
| 缓存 | Redis (go-redis) + 内存 |
| 认证 | JWT, WebAuthn/Passkeys, OAuth 2.0 (7 种) |
| 前端 | React 19, TypeScript, Rsbuild, Radix UI, Tailwind |
| 支付 | Stripe, Creem, Waffo |

## 统计

- 分析覆盖 Go 文件：~557 个（不含 `web/`，`find . -name "*.go"` 实测）
- 核心包：15 个（`common` / `constant` / `controller` / `dto` / `i18n` / `logger` / `middleware` / `model` / `oauth` / `pkg` / `relay` / `router` / `service` / `setting` / `types`）
- 渠道适配器：37 个（`relay/channel/` 一级子目录）；其中 OpenAI 兼容渠道（Azure、OpenRouter、Xinference、Moonshot、Submodel、SiliconFlow 等）复用 `openai/` 适配器，在 `constant/channel.go` 中独立分配 `ChannelType*` 常量（共 48 项）
- 中间件：21 个文件（`middleware/*.go`）
- 数据表：25 张（24 张通过 `migrateDBFast` 并发迁移 + 1 张 `SubscriptionPlan` 因 SQLite 兼容单独迁移）
- 分析文档总字数：约 25000 字

<!-- @end-section -->

<!-- @section: related -->
## 相关目录

- [[../claude/index|Claw Code 分析索引]] — Agent 运行时参考项目

<!-- @end-section -->
