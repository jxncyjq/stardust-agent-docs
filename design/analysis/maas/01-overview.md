---
id: "analysis-newapi-overview-001"
title: "New API 项目架构总览"
aliases: ["new-api overview", "新API网关概览"]
type: "analysis"
category: "design/analysis/maas"
tags: ["new-api", "architecture", "go", "llm-gateway", "maas"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: null
children:
  - "analysis-newapi-modules-002"
  - "analysis-newapi-channel-003"
  - "analysis-newapi-billing-004"
  - "analysis-newapi-middleware-005"
  - "analysis-newapi-datamodels-006"
  - "analysis-newapi-insights-007"
related_docs:
  - id: "analysis-newapi-modules-002"
    relation: "related_to"
    path: "./02-module-analysis.md"
  - id: "analysis-newapi-channel-003"
    relation: "related_to"
    path: "./03-channel-adapter-system.md"
  - id: "analysis-newapi-billing-004"
    relation: "related_to"
    path: "./04-billing-quota-system.md"
  - id: "analysis-newapi-middleware-005"
    relation: "related_to"
    path: "./05-middleware-and-flow.md"
  - id: "analysis-newapi-datamodels-006"
    relation: "related_to"
    path: "./06-data-models.md"
  - id: "analysis-newapi-insights-007"
    relation: "related_to"
    path: "./07-maas-insights.md"
---

<!-- @section: overview -->
# New API 项目架构总览

## 项目概述

New API 是**下一代 LLM 网关与 AI 资产管理系统**，对外提供统一的 OpenAI 兼容 API，内置用户管理、计费、速率限制和管理后台。当前版本暴露 48 个 `ChannelType*` 常量（OpenAI、Claude、Gemini、Azure、AWS Bedrock、ZhipuAI、Moonshot 等，见 `constant/channel.go`），底层由 37 个渠道适配器目录支持（`relay/channel/`），其中所有 OpenAI 兼容协议的渠道共享 `openai/` 适配器实现。

- 仓库地址：`Calcium-Ion/new-api`
- 主要语言：Go 1.22+（后端）/ TypeScript + React 19（前端）
- 框架：Gin（HTTP）、GORM（ORM）
- 数据库：SQLite / MySQL 5.7.8+ / PostgreSQL 9.6+ 三选一
- 缓存：Redis + 内存缓存

## 系统定位

```
外部用户/应用
    │  OpenAI 兼容 API (Bearer Token)
    ▼
┌──────────────────────────────────────────────┐
│              New API (网关层)                  │
│                                              │
│  Auth → RateLimit → Distributor → Relay      │
│                                              │
│  ┌────────┐ ┌────────┐ ┌────────┐                  │
│  │ OpenAI │ │ Claude │ │ Gemini │ ... 37 个适配器  │
│  └────────┘ └────────┘ └────────┘                  │
│       │         │         │                  │
│       ▼         ▼         ▼                  │
│  api.openai  api.anthropic  generativelang... │
└──────────────────────────────────────────────┘
```

## 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| Backend | Go 1.22+, Gin, GORM v2 | REST API + 中继代理 |
| Frontend | React 19, TypeScript, Rsbuild, Radix UI, Tailwind | 管理后台 |
| Database | SQLite / MySQL / PostgreSQL | GORM 抽象，三库兼容 |
| Cache | Redis (go-redis) + 内存 | 渠道缓存、用户缓存 |
| Auth | JWT, WebAuthn/Passkeys, OAuth 2.0 | 多方式登录 |
| i18n | go-i18n (后端) + i18next (前端) | 中英文 + 多语言 |

## 分层架构

```
router/        — HTTP 路由（API、Dashboard、Relay、Web）
    │
    ▼
middleware/    — 中间件链（Auth、RateLimiter、Distributor、Logger）
    │
    ▼
controller/    — 请求处理器（relay、user、channel、billing、token...）
    │
    ▼
service/       — 业务逻辑（channel_select、billing、quota、convert...）
    │
    ▼
model/         — 数据模型与 DB 访问（GORM）
    │
    ▼
relay/         — AI API 中继代理（含 40+ 提供商适配器）
  relay/channel/ — 各提供商专用适配器
```

## 核心数据流

```
用户请求 (Bearer Token + Model Name)
  │
  ▼
Auth 中间件 — 验证 Token，提取用户/分组信息
  │
  ▼
RateLimit 中间件 — 按用户/模型/IP 限流
  │
  ▼
Distributor 中间件 — 选择渠道 (Channel Selection)
  │  ├── Token 指定渠道 → 直接用
  │  ├── Token 模型限制检查
  │  ├── 分组优先级匹配
  │  └── 渠道健康检查 + 负载均衡
  │
  ▼
Controller Relay — 按 RelayMode 分发
  │  ├── Text → relay.TextHelper
  │  ├── Image → relay.ImageHelper
  │  ├── Audio → relay.AudioHelper
  │  └── Embedding → relay.EmbeddingHelper
  │
  ▼
Channel Adaptor — 格式转换 + HTTP 请求
  │  ├── 请求转换（OpenAI ↔ 各提供商格式）
  │  ├── SSE 流代理
  │  └── 响应转换（各提供商 → OpenAI 统一格式）
  │
  ▼
Quota 结算 — 扣除配额、记录日志
```

## 文档索引

- [[02-module-analysis|Go 模块功能分析]]
- [[03-channel-adapter-system|渠道适配器系统分析]]
- [[04-billing-quota-system|计费与配额系统分析]]
- [[05-middleware-and-flow|中间件与请求流程分析]]
- [[06-data-models|核心数据模型分析]]
- [[07-maas-insights|MaaS 洞察与 Legion 参考]]

<!-- @end-section -->
