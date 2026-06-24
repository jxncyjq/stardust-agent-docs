---
id: "analysis-sub2api-index-001"
title: "Sub2API 分析文档索引"
type: "index"
category: "design/analysis"
tags: ["sub2api", "analysis", "index"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "analysis-sub2api-overview-001"
    relation: "related_to"
    path: "./01-overview.md"
---

# Sub2API 分析文档索引

本目录包含对 Sub2API 代码库的全面架构、流程和代码分析。

## 文档列表

| 文件 | 内容 |
|------|------|
| [01-overview.md](./01-overview.md) | 项目概览、技术栈、目录结构 |
| [02-architecture.md](./02-architecture.md) | 整体架构、模块职责、数据流向 |
| [03-database.md](./03-database.md) | 数据库设计、35 个表的结构和字段 |
| [04-core-flows.md](./04-core-flows.md) | 核心业务流程、故障转移、并发控制 |
| [05-api-routes.md](./05-api-routes.md) | API 路由、中间件系统、前端路由 |
| [06-configuration.md](./06-configuration.md) | 配置系统、环境变量、Docker 部署 |

## 快速摘要

**Sub2API** 是一个 AI API 网关平台，将 AI 产品订阅（Claude、OpenAI、Gemini 等）转换为标准 API 服务。

核心能力：
- 多平台 API 转发（Anthropic、OpenAI、Google Gemini、Antigravity）
- 负载感知账号调度 + 粘性会话
- 故障转移（同账号重试 3 次 + 账号切换）
- Token 级别计费和余额管理
- 多种支付方式（支付宝、微信、Stripe、EasyPay）
- 多种 OAuth 登录（LinuxDo、WeChat、OIDC、GitHub、Google）
- Redis 有序集合实现的并发控制
- 仪表盘预聚合（90 天用量日志保留）

技术栈：Go 1.25+ / Gin / Ent ORM / PostgreSQL / Redis / Vue 3 / Vite

分析日期：2026-05-09
