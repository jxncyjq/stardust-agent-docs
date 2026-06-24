---
id: "analysis-newapi-middleware-005"
title: "中间件与请求流程分析"
aliases: ["new-api middleware", "请求流程", "middleware pipeline", "request lifecycle"]
type: "analysis"
category: "design/analysis/maas"
tags: ["new-api", "middleware", "request-flow", "auth", "rate-limit", "maas"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: "analysis-newapi-overview-001"
related_docs:
  - id: "analysis-newapi-overview-001"
    relation: "parent"
    path: "./01-overview.md"
  - id: "analysis-newapi-modules-002"
    relation: "related_to"
    path: "./02-module-analysis.md"
  - id: "analysis-newapi-channel-003"
    relation: "related_to"
    path: "./03-channel-adapter-system.md"
  - id: "analysis-newapi-billing-004"
    relation: "related_to"
    path: "./04-billing-quota-system.md"
---

<!-- @section: overview -->
# 中间件与请求流程分析

## 系统概述

New API 的请求处理基于 Gin 中间件链构建，形成完整的请求生命周期管道。请求从进入系统到返回响应，经历**认证 → 限流 → 分发 → 中继 → 结算**五个核心阶段，每个阶段由独立中间件或控制器负责。

## 全局中间件链（所有请求）

在 `main.go` 中，以下中间件按顺序应用于**所有路由**：

```
1. gin.CustomRecovery  → 全局 panic 捕获
2. RequestId()         → 生成唯一请求 ID
3. PoweredBy()         → 版本头 X-New-Api-Version
4. I18n()              → 语言检测
5. SetUpLogger()       → 请求日志格式化
6. Sessions()          → Session 存储（Cookie, 30天有效）
```

<!-- @end-section -->

<!-- @section: relay-lifecycle -->
## Relay 请求完整生命周期

以最常见的 **`POST /v1/chat/completions`** 为例：

### 中间件链顺序

```
CORS (全局)
  ↓
DecompressRequestMiddleware
  ├── gzip/brotli 解压
  └── MaxBytes 限制 (默认 32MB)
  ↓
BodyStorageCleanup          ← defer: 请求结束后清理临时文件
  ↓
StatsMiddleware             ← 活跃连接计数 (atomic)
  ↓
RouteTag("relay")           ← 设置 route_tag 用于日志
  ↓
SystemPerformanceCheck
  ├── CPU > 阈值 → 503
  ├── 内存 > 阈值 → 503
  └── 磁盘 > 阈值 → 503
  ↓
TokenAuth                   ← ★ 核心认证中间件
  ↓
ModelRequestRateLimit       ← 模型级请求限流
  ↓
Distribute                  ← ★ 渠道分配
  ↓
controller.Relay            ← 核心处理逻辑
```

### 阶段 1: 请求预处理

**DecompressRequestMiddleware** (`middleware/gzip.go`):
- Content-Encoding `gzip` → gzip 解压
- Content-Encoding `br` → brotli 解压
- 所有请求施加 `http.MaxBytesReader` 大小限制
- 替换 `c.Request.Body` 为解压后的 reader

**BodyStorageCleanup** (`middleware/body_cleanup.go`):
- `c.Next()` 之后执行（响应完成后）
- 清理 BodyStorage（内存/磁盘缓存）和文件源
- 确保请求体临时资源被释放

### 阶段 2: 系统保护

**SystemPerformanceCheck** (`middleware/performance.go`):
- 从 `PerformanceMonitorConfig` 读取阈值
- CPU/内存/磁盘任一超限 → 返回 503
- `/v1/messages` 路径返回 Claude 格式错误，其他返回 OpenAI 格式

### 阶段 3: 认证 — TokenAuth

**TokenAuth** (`middleware/auth.go`) 是核心认证中间件，执行多步验证：

#### 3.1 密钥提取（多协议兼容）

| 协议 | 提取方式 |
|------|----------|
| WebSocket | `Sec-WebSocket-Protocol: openai-insecure-api-key.xxx` |
| Anthropic | `x-api-key` 头 |
| Gemini | Query `key` 参数 或 `x-goog-api-key` 头 |
| 通用 | `Authorization: Bearer sk-xxx`，去 `sk-` 前缀，取 `-` 分隔第一段 |
| Midjourney | `mj-api-secret` 头 |

#### 3.2 令牌验证

```go
token, err := model.ValidateUserToken(key)
```
- 查 Redis 缓存 → DB 回退
- 检查令牌状态（禁用）/ 过期时间 / 剩余额度
- 额度耗尽令牌自动标记为禁用

#### 3.3 安全检查

- **IP 白名单**: 令牌配置了 `AllowIps` → 检查客户端 IP 是否在 CIDR 列表中
- **用户状态**: 通过 `model.GetUserCache` 检查用户是否被禁用
- **模型限制**: 令牌配置了 `ModelLimitsEnabled` → 检查请求模型是否在白名单中

#### 3.4 向下游传递的 Context 值

| Context Key | 含义 |
|-------------|------|
| `id` | 用户 ID |
| `token_id` | 令牌 ID |
| `token_key` | 令牌密钥 |
| `token_name` | 令牌名称 |
| `token_unlimited_quota` | 是否无限额度 |
| `token_model_limit_enabled` | 是否启用模型限制 |
| `token_model_limit` | 允许的模型列表 |
| `group` (usingGroup) | 当前使用的分组 |
| `token_group` | 令牌绑定的分组 |
| `token_cross_group_retry` | 跨分组重试开关 |
| `specific_channel_id` | 管理员指定渠道 ID |

### 阶段 4: 限流 — ModelRequestRateLimit

**ModelRequestRateLimit** (`middleware/model-rate-limit.go`):

**两层限流**:

| 级别 | Key | 算法 | 说明 |
|------|-----|------|------|
| 成功请求 | `MRRLS:{userId}` | 滑动窗口 | 仅限制成功请求 (status < 400) |
| 总请求 | `rateLimit:{userId}` | 令牌桶 | 限制所有请求 |

- **存储**: Redis 可用时用 Redis，否则用内存限流器
- **分组配置**: 支持通过 `GetGroupRateLimit(group)` 获取分组级别配置
- **可配置参数**: 启用开关、时间窗口、请求数限制

### 阶段 5: 分发 — Distribute

**Distribute** (`middleware/distributor.go`) 是路由决策核心：

#### 5.1 模型名提取

按请求路径和内容提取模型名：
- JSON 请求: 解析 `model` 字段
- Gemini: 从 `/v1beta/models/{model}:{action}` 路径提取
- Midjourney/Suno/视频生成: 特殊路由处理
- 图像/音频: 兜底默认值

#### 5.2 渠道选择（优先级递减）

```
a) 管理员指定渠道 (specific_channel_id)
   → 直接获取 Channel，跳过选择逻辑

b) 渠道亲和力 (Channel Affinity)
   → 检查用户是否对某渠道有缓存亲和力

c) 随机选择 (CacheGetRandomSatisfiedChannel)
   → 按 group + model + priority → 加权随机
```

#### 5.3 auto 分组跨分组重试

当 `token_group = "auto"` 且启用 `cross_group_retry` 时：

```
Retry=0: GroupA priority0 → 随机选择渠道
Retry=1: GroupA priority1 → 当前分组下一优先级
Retry=2: GroupA 用尽 → GroupB priority0
Retry=3: GroupB priority1
...
```

实现细节：
- `ContextKeyAutoGroupIndex` — 当前分组索引
- `ContextKeyAutoGroupRetryIndex` — 当前分组开始时的全局重试次数
- `priorityRetry = Retry - startRetryIndex` — 分组内优先级

#### 5.4 分发后 Context 值

| Context Key | 含义 |
|-------------|------|
| `original_model` | 原始模型名 |
| `channel_id` | 选中渠道 ID |
| `channel_type` | 渠道类型 |
| `channel_key` | 渠道 API Key（多 Key 轮询/随机） |
| `base_url` | 渠道基础 URL |
| `model_mapping` | 模型名映射 JSON |
| `param_override` | 参数覆盖 JSON |
| `header_override` | 请求头覆盖 JSON |
| `auto_ban` | 是否自动禁用 |
| `relay_mode` | 中继模式 |
| `request_start_time` | 请求开始时间 |

<!-- @end-section -->

<!-- @section: controller-relay -->
## Controller.Relay 核心处理

### 整体流程

```
1. 获取 requestId
2. WebSocket 升级（如果是 realtime 格式）
3. 解析和验证请求 (helper.GetAndValidateRequest)
4. 生成 RelayInfo 对象
5. 敏感词检查（如果启用）
6. 估算输入 Token 数
7. 计算价格数据 (helper.ModelPriceHelper)
8. 预扣费 (service.PreConsumeBilling)
9. 设置 defer 退款（失败时退还）
10. ╔══ 重试循环 ══╗
    ║ 获取渠道        ║
    ║ 执行 Relay       ║
    ║ 成功 → 退出循环  ║
    ║ 失败 → 处理错误  ║
    ║ 判断是否重试     ║
    ╚═════════════════╝
11. 结算 (service.SettleBilling)
12. 记录日志
```

### 重试决策

**跳过重试的条件**:
- 渠道亲和力失败
- 错误标记 `skipRetry`
- 剩余重试次数 <= 0
- 管理员指定了特定渠道
- 状态码在 200-299（成功）
- 状态码 < 100 或 > 599 → 重试
- 配置中始终跳过的错误码
- `operation_setting.ShouldRetryByStatusCode` 判断

### 渠道错误处理

1. **自动禁用**: `ShouldDisableChannel` 返回 true 且渠道启用 `autoBan` → 异步禁用
2. **错误日志**: 记录到 DB（用户 ID、渠道 ID、模型名、耗时等）
3. **重试**: 重新进入渠道选择

### 结算

```
delta = actualQuota - preConsumedQuota
├── delta > 0 → 补扣资金 + 令牌额度
├── delta < 0 → 退还多余预扣
└── 发送额度通知
```

<!-- @end-section -->

<!-- @section: api-middleware -->
## API 路由中间件

### API 路由基础中间件

```
RouteTag("api")
  → gzip.Gzip (响应压缩)
  → BodyStorageCleanup
  → GlobalAPIRateLimit
```

### 认证中间件层级

| 中间件 | 认证方式 | 最低角色 | 应用路由 |
|--------|----------|----------|----------|
| `UserAuth()` | Session Cookie + Access Token | CommonUser | 用户自身资源 |
| `AdminAuth()` | Session Cookie | AdminUser | 管理操作 |
| `RootAuth()` | Session Cookie | RootUser | 系统配置 |
| `TokenAuth()` | Bearer Token / API Key | — | AI API 代理 |
| `TokenAuthReadOnly()` | 宽松 Token 验证 | — | 令牌用量查询 |

### 专项限流中间件

| 中间件 | 限制 | 应用场景 |
|--------|------|----------|
| `CriticalRateLimit` | 20次/20分钟 | 登录、注册、OAuth、充值 |
| `EmailVerificationRateLimit` | 2次/30秒 | 邮件验证码发送 |
| `SearchRateLimit` | 10次/60秒(按用户ID) | Token/Log 搜索 |
| `TurnstileCheck` | Cloudflare 人机验证 | 注册、登录、签到 |
| `DisableCache` | 禁止浏览器缓存 | Token/Channel Key 查询 |
| `SecureVerificationRequired` | 二次安全验证(5分钟有效期) | Channel Key 查看 |

<!-- @end-section -->

<!-- @section: context-flow -->
## Context 数据传递全景图

### 核心 Context Key 定义

```go
// 令牌相关
"token_unlimited_quota" / "token_key" / "token_id" / "token_group"
"specific_channel_id" / "token_model_limit_enabled" / "token_model_limit"

// 渠道相关
"channel_id" / "channel_type" / "channel_key" / "base_url"
"model_mapping" / "status_code_mapping" / "auto_ban"
"param_override" / "header_override"

// 用户相关
"id" / "user_group" / "group" / "username" / "user_setting"

// 请求相关
"original_model" / "request_start_time" / "is_stream"
"language" / "prompt_tokens" / "estimated_tokens"
```

### 数据流向

```
[RequestId] → "X-Oneapi-Request-Id"
[PoweredBy] → "X-New-Api-Version"
[I18n]      → "language"
[Session]   → "username", "role", "id", "status", "group"
[TokenAuth] → "id", "token_id", "token_key", "token_group",
               "token_model_limit", "group"(usingGroup), ...
[Distribute]→ "channel_id", "channel_type", "channel_key",
               "base_url", "model_mapping", "relay_mode", ...
[Relay]     → 读取所有 context → 执行 relay → 记录结果
[Cleanup]   → 请求结束后清理 BodyStorage
```

### 数据传递机制

- 使用 `gin.Context.Set(key, value)` 写入
- 使用 `common.SetContextKey(c, key, value)` 封装（同时写入 Request Context）
- 下游通过 `c.Get(key)` 或 `common.GetContextKey()` 读取
- `common.GetContextKeyString/Bool/Int` 提供类型安全读取

<!-- @end-section -->

<!-- @section: rate-limit -->
## 限流系统总结

### 限流类型与实现

| 限流类型 | Key 格式 | 算法 | 存储 |
|----------|----------|------|------|
| GlobalWebRateLimit | `rateLimit:GW:{IP}` | 滑动窗口 | Redis/内存 |
| GlobalAPIRateLimit | `rateLimit:GA:{IP}` | 滑动窗口 | Redis/内存 |
| CriticalRateLimit | `rateLimit:CT:{IP}` | 滑动窗口 | Redis/内存 |
| ModelRequestRateLimit(总) | `rateLimit:{userId}` | 令牌桶 | Redis/内存 |
| ModelRequestRateLimit(成功) | `MRRLS:{userId}` | 滑动窗口 | Redis/内存 |
| SearchRateLimit | `{mark}:user:{userId}` | 按用户ID | Redis/内存 |
| EmailVerificationRateLimit | `emailVerification:EV:{IP}` | 计数器+TTL | Redis/内存 |

### 内存限流器实现

`common/rate-limit.go` — `InMemoryRateLimiter`:
- 数据结构: `map[string]*[]int64`（Key → 时间戳队列）
- 算法: 每次请求将当前时间戳加入队列，删除过期条目，判断是否超限
- 后台协程定期清理过期 Key

### Redis 限流器

- 滑动窗口: Redis List（`LPush`/`LTrim`/`LIndex`）
- 令牌桶: `common/limiter/` 包实现

<!-- @end-section -->

<!-- @section: body-storage -->
## 请求体存储系统

为支持多次读取请求体（中间件解析模型名 + controller relay 重试），实现了 `BodyStorage` 接口：

```
BodyStorage (接口)
  ├── memoryStorage: 小请求体（< 阈值），存内存
  └── diskStorage: 大请求体，写临时文件
```

- `UnmarshalBodyReusable`: 解析后自动 Seek 回开头
- `BodyStorageCleanup`: 请求结束后自动关闭并清理
- 大请求体阈值由磁盘缓存配置决定

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|New API 项目架构总览]]
- [[02-module-analysis|Go 模块功能分析]]
- [[03-channel-adapter-system|渠道适配器系统分析]]
- [[04-billing-quota-system|计费与配额系统分析]]
- [[06-data-models|核心数据模型分析]]

<!-- @end-section -->
