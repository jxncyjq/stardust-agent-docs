---
id: "spec-component-rate-limiter-012"
title: "RateLimiter 组件规范"
aliases: ["RateLimiter规范", "速率限制器", "rate-limiter-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "rate-limit", "pipeline", "redis", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C12"
layer: "L2"
depends_on: []
optional_deps:
  - "C06"   # ConcurrencyLimiter — 可选结合使用做并发保护
conflicts_with: []
required_by: []
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# RateLimiter 组件规范

## 1. 组件定位

`RateLimiter` 在请求管道的早期（TenantContextLoader 之后）对**每个用户 / 用户组**的请求速率（RPM/RPS）做限制，防止单个调用方占用过多资源。

与 `QuotaBudgetManager`（C33）的区别：
- `RateLimiter`：关注**请求频率**（每分钟多少次），基于时间滑动窗口
- `QuotaBudgetManager`：关注**消耗总量**（一个月用了多少 token），基于配额预算

```
AuthGate → TenantContextLoader → RateLimiter → QuotaChecker → ...
                                      │
                                      │ 超速 → 429 Too Many Requests
                                      │ 含 Retry-After 头
```

**读者**：配置速率限制策略的运维工程师、实现限流的开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// RateLimiter 对请求速率（RPM/RPS）做限制。
// 并发安全：所有方法可被多个 goroutine 并发调用。
type RateLimiter interface {
    // Allow 检查并计数当前请求是否在速率限制内。
    // key 通常是 userID 或 groupID（框架从 RequestContext 中提取）。
    // 允许时返回 nil；超速时返回 *RateLimitError（含 Retry-After）。
    Allow(ctx context.Context, key string) error

    // Peek 仅检查不计数（用于预检，不消耗配额）。
    Peek(key string) (remaining int, resetAt time.Time, err error)
}

// RateLimitError 超速时返回的错误。
type RateLimitError struct {
    Key        string
    Limit      int
    Window     time.Duration
    RetryAfter time.Duration // 建议等待时间
}
```

<!-- @end-section -->

<!-- @section: algorithm -->
---

## 3. 限流算法

### 3.1 固定窗口（默认，简单高效）

```
每个时间窗口（如每分钟）内允许 N 次请求。
Redis 实现：INCR key:{userID}:{minute_bucket}，EXPIRE key 60。
缺点：窗口边界可能允许 2× 突发（窗口末 N 次 + 下窗口初 N 次）。
```

### 3.2 滑动窗口（更精确）

```
使用 Redis ZSET 记录请求时间戳，仅统计最近 W 秒内的请求数。
Lua 脚本保证原子性：
  ZREMRANGEBYSCORE key 0 (now - window)
  count = ZCARD key
  if count >= limit: reject
  ZADD key now now
  EXPIRE key window
```

### 3.3 令牌桶（支持突发）

允许短时突发，但长期速率受限：

```
capacity = limit（桶容量）
fill_rate = limit / window（填充速率）
Redis 存储 (last_tokens, last_refill_time)，原子操作更新。
```

<!-- @end-section -->

<!-- @section: config -->
---

## 4. 配置 Schema

```go
// RateLimiterConfig 是 RateLimiter 的配置。
type RateLimiterConfig struct {
    // Backend 限流存储后端。
    Backend string `default:"redis" validate:"oneof=redis memory"`
    RedisURL string

    // Algorithm 限流算法。
    Algorithm string `default:"fixed_window" validate:"oneof=fixed_window sliding_window token_bucket"`

    // DefaultRPM 未配置用户的默认每分钟请求限制（0 = 不限）。
    DefaultRPM int `default:"60"`

    // DefaultRPS 未配置用户的默认每秒请求限制（0 = 不限）。
    DefaultRPS int `default:"0"`

    // KeyFunc 从 RequestContext 提取限流 key 的函数（默认按 userID）。
    // 可配置为 groupID、tenantID 等。
    KeyFunc string `default:"user_id" validate:"oneof=user_id group_id tenant_id"`

    // OverLimitHeader 超速时响应头中包含的重试时间头名称。
    RetryAfterHeader string `default:"Retry-After"`
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 5. 行为契约

| 契约 | 说明 |
|------|------|
| **Allow 原子操作** | 检查和计数必须是原子操作（Lua 脚本保证） |
| **超速返回 Retry-After** | RetryAfter 必须是合理的等待时间（非零） |
| **Peek 不计数** | Peek 只读，不消耗速率配额 |
| **Backend 不可用时降级** | Redis 故障时可配置为允许所有请求（不限流）或拒绝 |

<!-- @end-section -->

<!-- @section: checklist -->
---

## 6. 实现检查清单

```
RateLimiter
  ☐ Allow：原子检查 + 计数（Lua 脚本）
  ☐ 超速时返回含 RetryAfter 的 RateLimitError
  ☐ Peek：只读，不消耗
  ☐ Backend 不可用时的降级策略（可配置）
  ☐ 支持按 user_id / group_id / tenant_id 限流

测试
  ☐ 并发 Allow 不超额（竞态测试）
  ☐ 窗口重置后恢复
  ☐ Redis 不可用降级行为
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C12 在依赖图中的位置）
- concurrency-limiter-spec.md（C06，可选结合）
- quota-budget-manager-spec.md（C33，配额总量管理）

<!-- @end-section -->
