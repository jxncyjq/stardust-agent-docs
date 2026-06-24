---
id: "analysis-sub2api-overview-001"
title: "Sub2API 项目概览"
type: "analysis"
category: "design/analysis"
tags: ["sub2api", "overview", "analysis"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "analysis-sub2api-index-001"
    relation: "related_to"
    path: "./README.md"
---

# Sub2API 项目概览

## 项目定义

**Sub2API** 是一个 AI API 网关平台，用于分发和管理 AI 产品订阅的 API 配额。用户通过平台生成的 API Key 调用上游 AI 服务，平台负责鉴权、计费、负载均衡和请求转发。

官方域名：`sub2api.org` 和 `pincc.ai`

## 技术栈

| 组件 | 技术 | 版本 |
|------|------|------|
| 后端语言 | Go | 1.25.7+ |
| Web 框架 | Gin | 1.9.1 |
| ORM | Ent | 0.14.5 |
| 前端框架 | Vue | 3.4+ |
| 前端构建 | Vite | 5.0+ |
| 样式 | TailwindCSS | 3.4+ |
| 数据库 | PostgreSQL | 15+ |
| 缓存/队列 | Redis | 7+ |
| 容器化 | Docker | Ready |

## 项目类型

- **微服务网关** — API 请求转发和负载均衡
- **SaaS 平台** — 多租户用户管理和计费系统
- **支付系统** — 内置支付处理（支付宝、微信、Stripe、EasyPay）

## 支持的上游平台

| 平台 | API 类型 |
|------|---------|
| Anthropic | Claude Messages API |
| OpenAI | Chat Completions、Responses API |
| Google | Gemini v1beta API |
| Antigravity | 混合 Claude/Gemini |
|  Bedrock | 通过凭证转发 |
| Google Vertex AI | 通过凭证转发 |

## 目录结构

```
sub2api/
├── backend/                    # Go 后端
│   ├── cmd/server/main.go      # 入口点
│   ├── internal/
│   │   ├── handler/            # HTTP 请求处理层
│   │   ├── service/            # 业务逻辑层（274 个文件）
│   │   ├── repository/         # 数据访问层
│   │   ├── domain/             # 业务常量和枚举
│   │   ├── config/             # 配置管理（2766 行）
│   │   ├── server/
│   │   │   ├── middleware/     # HTTP 中间件
│   │   │   └── routes/         # 路由注册
│   │   ├── payment/            # 支付系统
│   │   ├── integration/        # 第三方集成
│   │   └── pkg/                # 通用工具库
│   └── ent/                    # ORM 生成代码（35 个表）
│       └── schema/             # 数据库 Schema 定义
├── frontend/                   # Vue 3 前端
│   └── src/
│       ├── views/              # 页面组件
│       ├── components/         # 可复用组件
│       ├── stores/             # Pinia 状态管理
│       ├── api/                # API 调用封装
│       ├── router/             # 路由配置
│       ├── utils/              # 工具函数
│       ├── types/              # TypeScript 类型定义
│       └── i18n/               # 国际化
└── deploy/                     # 部署配置
    ├── docker-compose.yml
    └── Dockerfile
```
