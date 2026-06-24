---
id: "analysis-sub2api-architecture-001"
title: "Sub2API 架构分析"
type: "analysis"
category: "design/analysis"
tags: ["sub2api", "architecture", "analysis"]
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

# Sub2API 架构分析

## 整体架构

Sub2API 采用前后端分离的分层架构，后端为 Go + Gin 的 RESTful API 服务，前端为 Vue 3 SPA。

```
┌─────────────────────────────────────────────────────────────┐
│                    前端 (Vue 3 + Vite)                       │
│  ├─ 用户面板 (Dashboard)                                     │
│  ├─ 管理后台 (Admin Panel)                                   │
│  ├─ 认证页面 (Auth Pages)                                    │
│  └─ 设置页面 (Settings)                                      │
└────────────────────┬────────────────────────────────────────┘
                     │ HTTP/2 Cleartext (h2c)
┌────────────────────▼────────────────────────────────────────┐
│              后端 API 网关 (Go + Gin)                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 中间件层 (Middleware)                                 │   │
│  │ ├─ RequestLogger / Logger                            │   │
│  │ ├─ CORS / SecurityHeaders / CSP                      │   │
│  │ ├─ JWTAuth / APIKeyAuth / AdminAuth                  │   │
│  │ └─ Recovery / RequestBodyLimit                       │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 路由层 (Routes)                                       │   │
│  │ ├─ /api/v1/auth/*     认证相关                       │   │
│  │ ├─ /api/v1/user/*     用户管理                       │   │
│  │ ├─ /api/v1/admin/*    管理功能                       │   │
│  │ ├─ /api/v1/payment/*  支付处理                       │   │
│  │ ├─ /v1/messages       Claude API 转发               │   │
│  │ ├─ /v1/chat/completions  OpenAI 兼容               │   │
│  │ ├─ /v1beta/models     Gemini API                    │   │
│  │ └─ /v1/responses      OpenAI Responses             │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 业务逻辑层 (Services)                                 │   │
│  │ ├─ GatewayService      网关核心                      │   │
│  │ ├─ AccountService      账号管理                      │   │
│  │ ├─ APIKeyService        API Key 管理                 │   │
│  │ ├─ UserService          用户管理                     │   │
│  │ ├─ SubscriptionService  订阅管理                     │   │
│  │ ├─ BillingCacheService  计费缓存                     │   │
│  │ ├─ UsageService         用量统计                     │   │
│  │ ├─ PaymentService       支付处理                     │   │
│  │ ├─ AuthService          认证服务                     │   │
│  │ └─ ConcurrencyService   并发控制                     │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 数据访问层 (Repository)                               │   │
│  │ ├─ AccountRepository                                 │   │
│  │ ├─ APIKeyRepository                                  │   │
│  │ ├─ UserRepository                                    │   │
│  │ └─ BillingCacheRepository                            │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
┌───────▼──┐  ┌──────▼──┐  ┌─────▼──────┐
│PostgreSQL│  │  Redis  │  │ 上游 API   │
│ 数据库   │  │ 缓存/队列 │  │ (Claude等) │
└──────────┘  └─────────┘  └────────────┘
```

## 模块职责

### 后端核心模块

| 模块 | 路径 | 职责 |
|------|------|------|
| Handler | `internal/handler/` | HTTP 请求处理，路由映射 |
| Service | `internal/service/` | 业务逻辑实现（274 个文件） |
| Repository | `internal/repository/` | 数据库访问层 |
| Domain | `internal/domain/` | 业务常量和枚举 |
| Config | `internal/config/` | 配置管理 |
| Middleware | `internal/server/middleware/` | HTTP 中间件 |
| Payment | `internal/payment/` | 支付系统 |
| Integration | `internal/integration/` | 第三方集成 |
| Pkg | `internal/pkg/` | 通用工具库 |
| Ent | `ent/` | ORM 生成代码 |

### 前端核心模块

| 模块 | 路径 | 职责 |
|------|------|------|
| Views | `src/views/` | 页面组件 |
| Components | `src/components/` | 可复用组件 |
| Stores | `src/stores/` | Pinia 状态管理 |
| API | `src/api/` | API 调用封装 |
| Router | `src/router/` | 路由配置 |
| Utils | `src/utils/` | 工具函数 |
| Types | `src/types/` | TypeScript 类型定义 |
| I18n | `src/i18n/` | 国际化 |

## 关键架构决策

### H2C（HTTP/2 Cleartext）
后端支持 H2C，允许在不使用 TLS 的情况下使用 HTTP/2，适合内网部署场景。

### 前端嵌入
前端构建产物通过 Go embed 嵌入到后端二进制中，简化部署（单二进制文件）。

### 连接池隔离策略
网关支持多种连接池隔离模式：
- `proxy` — 按代理隔离
- `account` — 按账号隔离
- `account_proxy` — 按账号+代理隔离

### Outbox 轮询机制
使用 Outbox 模式处理调度缓存，支持：
- 轮询周期配置
- 滞后告警和强制重建
- 积压重建阈值

## 数据流向

```
用户请求
  → API Key 认证
  → 用户/订阅验证
  → 请求规范化
  → 账号调度（负载感知 + 粘性会话）
  → 请求转发到上游 API
  → 响应处理
  → 计费（Token 计算 + 余额扣除）
  → 用量日志记录
  → 返回响应
```
