---
id: "analysis-sub2api-database-001"
title: "Sub2API 数据库设计"
type: "analysis"
category: "design/analysis"
tags: ["sub2api", "database", "schema"]
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

# Sub2API 数据库设计

## 概览

使用 PostgreSQL + Ent ORM，共 35 个数据库表。所有表均包含软删除（`deleted_at`）和时间戳（`created_at`、`updated_at`）。

## 实体分类

### 用户和认证

| 表 | 说明 |
|---|---|
| `User` | 用户账户（邮箱、密码、余额、并发数、TOTP 2FA） |
| `AuthIdentity` | 第三方认证身份（OAuth、OIDC、WeChat、LinuxDo） |
| `AuthIdentityChannel` | 认证渠道配置 |
| `PendingAuthSession` | OAuth 待处理会话 |

### 账号和分组

| 表 | 说明 |
|---|---|
| `Account` | AI API 账户（Claude、Gemini、OpenAI 等） |
| `AccountGroup` | 账号分组 |
| `Group` | 用户分组（订阅类型） |
| `UserAllowedGroup` | 用户允许的分组 |

### API Key 和访问控制

| 表 | 说明 |
|---|---|
| `APIKey` | 用户生成的 API Key |
| `SecuritySecret` | 安全密钥 |

### 计费和支付

| 表 | 说明 |
|---|---|
| `PaymentOrder` | 支付订单 |
| `PaymentAuditLog` | 支付审计日志 |
| `PaymentProviderInstance` | 支付提供商实例 |
| `PromoCode` | 促销码 |
| `PromoCodeUsage` | 促销码使用记录 |
| `RedeemCode` | 兑换码 |

### 订阅和配额

| 表 | 说明 |
|---|---|
| `UserSubscription` | 用户订阅 |
| `SubscriptionPlan` | 订阅计划 |

### 用量和监控

| 表 | 说明 |
|---|---|
| `UsageLog` | 用量日志（Token 级别） |
| `UsageCleanupTask` | 用量清理任务 |
| `ChannelMonitor` | 渠道监控 |
| `ChannelMonitorHistory` | 监控历史 |
| `ChannelMonitorDailyRollup` | 每日汇总 |
| `ChannelMonitorRequestTemplate` | 请求模板 |

### 系统配置

| 表 | 说明 |
|---|---|
| `Announcement` | 公告 |
| `AnnouncementRead` | 公告阅读记录 |
| `Setting` | 系统设置 |
| `Proxy` | 代理配置 |
| `TLSFingerprintProfile` | TLS 指纹配置 |
| `ErrorPassthroughRule` | 错误透传规则 |
| `IdentityAdoptionDecision` | 身份采纳决策 |
| `IdempotencyRecord` | 幂等性记录 |
| `UserAttributeDefinition` | 用户自定义属性定义 |
| `UserAttributeValue` | 用户自定义属性值 |

## 关键表字段

### User 表

```
email                    unique（软删除后可复用，partial index）
password_hash
balance                  decimal(20,8)
concurrency              并发数限制
role                     admin / user
status                   active / disabled
totp_secret_encrypted    2FA 密钥（加密存储）
signup_source            email / linuxdo / wechat / oidc / github / google
last_login_at
last_active_at
balance_notify_enabled
```

### Account 表

```
name, platform, type
credentials              JSONB — 存储 API Key、OAuth Token 等（加密）
extra                    JSONB — 平台特定数据
proxy_id                 代理配置
concurrency              账号级并发限制
load_factor              负载因子
rpm                      每分钟请求数限制
quota_reset_type         daily / weekly / monthly
status                   active / disabled / error
```

### APIKey 表

```
key                      unique
user_id, group_id
quota_limit, quota_used
rate_limit_5h            5 小时速率限制
rate_limit_1d            1 天速率限制
rate_limit_7d            7 天速率限制
ip_whitelist, ip_blacklist
expires_at
last_used_at
status                   active / disabled / expired
```

### UsageLog 表

```
user_id, api_key_id, account_id
endpoint                 规范化的 API 端点
input_tokens
output_tokens
cache_read_tokens
cost                     计费成本
created_at
```

## 数据库特性

- **软删除**：所有表有 `deleted_at` 字段（SoftDeleteMixin）
- **时间戳**：所有表有 `created_at` 和 `updated_at`（TimeMixin）
- **JSONB 支持**：Account 的 `credentials` 和 `extra` 字段使用 JSONB
- **部分索引**：软删除后支持唯一约束复用（`WHERE deleted_at IS NULL`）
- **连接池**：可配置的 PostgreSQL 连接池参数（max_open_conns、max_idle_conns 等）
