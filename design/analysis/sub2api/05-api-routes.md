---
id: "analysis-sub2api-api-routes-001"
title: "Sub2API API 路由和中间件"
type: "analysis"
category: "design/analysis"
tags: ["sub2api", "api", "routes", "middleware"]
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

# Sub2API API 路由和中间件

## 路由注册结构

```
SetupRouter()
  ├─ 应用全局中间件
  ├─ registerRoutes()
  │   ├─ RegisterCommonRoutes()      健康检查、状态
  │   ├─ RegisterAuthRoutes()        认证相关
  │   ├─ RegisterUserRoutes()        用户相关
  │   ├─ RegisterAdminRoutes()       管理员相关
  │   ├─ RegisterGatewayRoutes()     API 网关
  │   └─ RegisterPaymentRoutes()     支付相关
  └─ 嵌入前端静态文件
```

## API 网关端点

| 路由 | 方法 | 说明 |
|------|------|------|
| `/v1/messages` | POST | Claude Messages API |
| `/v1/messages/count_tokens` | POST | Token 计数 |
| `/v1/models` | GET | 模型列表 |
| `/v1/usage` | GET | 使用统计 |
| `/v1/responses` | POST/GET | OpenAI Responses API |
| `/v1/responses/*subpath` | POST | OpenAI Responses 子路径 |
| `/v1/chat/completions` | POST | OpenAI Chat Completions |
| `/v1/images/generations` | POST | 图片生成 |
| `/v1/images/edits` | POST | 图片编辑 |
| `/v1beta/models` | GET/POST | Gemini 模型 API |
| `/antigravity/v1/messages` | POST | Antigravity 专用路由 |
| `/responses` | POST/GET | 无前缀别名 |
| `/chat/completions` | POST | 无前缀别名 |
| `/backend-api/codex/responses` | POST/GET | Codex 直连 |

## 认证路由

| 路由 | 方法 | 限流 | 说明 |
|------|------|------|------|
| `/api/v1/auth/register` | POST | 5/min | 注册 |
| `/api/v1/auth/login` | POST | 20/min | 登录 |
| `/api/v1/auth/login/2fa` | POST | 20/min | 2FA 登录 |
| `/api/v1/auth/send-verify-code` | POST | 5/min | 发送验证码 |
| `/api/v1/auth/refresh` | POST | 30/min | 刷新 Token |
| `/api/v1/auth/logout` | POST | — | 登出 |
| `/api/v1/auth/validate-promo-code` | POST | 10/min | 验证优惠码 |
| `/api/v1/auth/oauth/*/start` | GET | — | OAuth 开始 |
| `/api/v1/auth/oauth/*/callback` | GET | — | OAuth 回调 |

## 用户路由（需 JWT 认证）

| 路由 | 方法 | 说明 |
|------|------|------|
| `/api/v1/user/profile` | GET/PUT | 用户信息 |
| `/api/v1/user/keys` | GET/POST | API Key 管理 |
| `/api/v1/user/keys/:id` | PUT/DELETE | API Key 操作 |
| `/api/v1/user/usage` | GET | 用量查询 |
| `/api/v1/user/balance` | GET | 余额查询 |
| `/api/v1/user/subscription` | GET | 订阅信息 |
| `/api/v1/user/payment/orders` | GET/POST | 支付订单 |
| `/api/v1/user/redeem` | POST | 兑换码 |
| `/api/v1/user/announcements` | GET | 公告列表 |

## 管理员路由（需 Admin 认证）

| 路由 | 方法 | 说明 |
|------|------|------|
| `/api/v1/admin/users` | GET/POST | 用户管理 |
| `/api/v1/admin/accounts` | GET/POST | 账号管理 |
| `/api/v1/admin/groups` | GET/POST | 分组管理 |
| `/api/v1/admin/settings` | GET/PUT | 系统设置 |
| `/api/v1/admin/promo-codes` | GET/POST | 促销码管理 |
| `/api/v1/admin/redeem-codes` | GET/POST | 兑换码管理 |
| `/api/v1/admin/announcements` | GET/POST | 公告管理 |
| `/api/v1/admin/proxies` | GET/POST | 代理配置 |
| `/api/v1/admin/channel-monitor` | GET | 渠道监控 |
| `/api/v1/admin/backup` | GET/POST | 备份管理 |

## 中间件系统

**文件**：`backend/internal/server/middleware/`

| 中间件 | 文件 | 功能 |
|------|------|------|
| `RequestLogger()` | request_logger.go | 请求日志 |
| `Logger()` | logger.go | 响应日志 |
| `CORS()` | cors.go | CORS 处理 |
| `SecurityHeaders()` | security_headers.go | CSP、安全头 |
| `JWTAuthMiddleware` | jwt_auth.go | JWT 认证 |
| `AdminAuthMiddleware` | admin_auth.go | 管理员认证 |
| `APIKeyAuthMiddleware` | api_key_auth.go | API Key 认证 |
| `APIKeyAuthWithSubscriptionGoogle()` | api_key_auth_google.go | Google API Key 认证 |
| `BackendModeAuthGuard()` | backend_mode_guard.go | 后端模式守卫 |
| `ClientRequestID()` | client_request_id.go | 请求 ID 生成 |
| `Recovery()` | recovery.go | 恐慌恢复 |
| `RequestBodyLimit()` | request_body_limit.go | 请求体大小限制 |
| `ForcePlatform()` | middleware.go | 强制平台设置 |
| `RequireGroupAssignment()` | middleware.go | 分组检查 |

### 关键中间件实现

**JWT 认证**：
- 从 `Authorization` 头提取 Bearer token
- 验证签名和过期时间
- 设置用户上下文

**API Key 认证**：
- 支持 `Authorization` 头和查询参数
- 缓存验证结果（减少数据库查询）
- 支持 Google API Key 特殊处理

**安全头**：
- CSP（Content-Security-Policy）
- 动态 nonce 注入
- frame-src 动态刷新

**分组检查**：
- 验证 API Key 是否分配到分组
- 支持协议特定的错误格式（Anthropic/Google）

## 前端路由

**文件**：`frontend/src/router/index.ts`

### 公开路由（无需认证）

| 路由 | 说明 |
|------|------|
| `/setup` | 设置向导 |
| `/home` | 首页 |
| `/login` | 登录 |
| `/register` | 注册 |
| `/email-verify` | 邮箱验证 |
| `/auth/callback` | OAuth 回调 |
| `/auth/linuxdo/callback` | LinuxDo OAuth |
| `/auth/wechat/callback` | 微信 OAuth |
| `/auth/oidc/callback` | OIDC OAuth |
| `/forgot-password` | 忘记密码 |
| `/reset-password` | 重置密码 |
| `/key-usage` | 密钥使用统计 |
| `/legal/:documentId` | 法律文档 |

### 用户路由（需认证）

| 路由 | 说明 |
|------|------|
| `/dashboard` | 仪表盘 |
| `/keys` | API 密钥管理 |
| `/usage` | 使用记录 |
| `/redeem` | 兑换码 |
| `/subscription` | 订阅管理 |
| `/account` | 账户设置 |
| `/announcement` | 公告 |
| `/channel-monitor` | 频道监控 |

### 管理员路由（需管理员权限）

| 路由 | 说明 |
|------|------|
| `/admin/dashboard` | 管理仪表盘 |
| `/admin/users` | 用户管理 |
| `/admin/accounts` | 账户管理 |
| `/admin/channels` | 频道管理 |
| `/admin/settings` | 系统设置 |
| `/admin/promo-codes` | 促销码 |
| `/admin/redeem-codes` | 兑换码 |
| `/admin/announcements` | 公告 |
| `/admin/proxies` | 代理配置 |
| `/admin/channel-monitor` | 渠道监控 |
| `/admin/backup` | 备份 |
| `/admin/risk-control` | 风险控制 |
