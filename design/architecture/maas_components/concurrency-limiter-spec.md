---
id: "spec-component-concurrency-limiter-006"
title: "ConcurrencyLimiter 组件规范"
aliases: ["ConcurrencyLimiter规范", "并发限制器", "concurrency-limiter-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "reliability", "concurrency", "rate-limit", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C06"
layer: "L1"
depends_on:
  - "C01"   # ModelProvider — 按提供商维度限制并发
optional_deps: []
conflicts_with: []
required_by:
  - "C12"   # RateLimiter 可选使用 ConcurrencyLimiter 做并发保护
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# ConcurrencyLimiter 组件规范

## 1. 组件定位

`ConcurrencyLimiter` 限制对每个 `ModelProvider` 的**最大并发请求数**，防止单个提供商被过多并发请求压垮，也防止框架自身因持有过多飞行中请求而耗尽资源。

它在请求进入 `ProviderExecutor` 调用上游前**获取令牌**，在请求完成后**释放令牌**。

```
ProviderExecutor
        │ 调用上游前
        ├── ConcurrencyLimiter.Acquire(providerID, ctx)
        │     如果并发已满：等待或拒绝（依配置）
        │
        │ [上游 API 调用]
        │
        └── ConcurrencyLimiter.Release(providerID)
              （defer 保证释放）
```

**读者**：配置并发策略的运维工程师、实现自定义限流的开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// ConcurrencyLimiter 控制对各提供商的最大并发飞行请求数。
// 并发安全：所有方法可被多个 goroutine 并发调用。
type ConcurrencyLimiter interface {
    // Acquire 请求一个并发槽位。
    // ctx 含超时：超时前未获得槽位时返回 ErrConcurrencyLimitExceeded。
    // WaitTimeout=0 时立即失败（不等待），WaitTimeout>0 时等待至超时。
    // 成功后调用方必须配对调用 Release（通常通过 defer）。
    Acquire(ctx context.Context, providerID string) error

    // Release 归还一个并发槽位。
    // 必须与 Acquire 配对调用；多余的 Release 调用视为无操作（不 panic）。
    Release(providerID string)

    // Stats 返回当前并发统计（用于监控和遥测）。
    Stats(providerID string) ConcurrencyStats
}

// ConcurrencyStats 并发状态快照。
type ConcurrencyStats struct {
    ProviderID  string
    MaxSlots    int     // 最大并发数（来自 ProviderConfig.ConcurrencyMax）
    ActiveSlots int     // 当前占用的槽位数
    WaitQueue   int     // 等待获取槽位的请求数
}
```

<!-- @end-section -->

<!-- @section: implementations -->
---

## 3. 内置实现

### 3.1 MemoryConcurrencyLimiter

基于内存的信号量实现，适用于单实例部署。

```go
// MemoryConcurrencyLimiter 使用 chan struct{} 实现每个提供商的信号量。
type MemoryConcurrencyLimiter struct {
    semaphores  map[string]chan struct{} // providerID → 信号量 channel
    waitTimeout time.Duration
    mu          sync.RWMutex
}

func (m *MemoryConcurrencyLimiter) Acquire(ctx context.Context, providerID string) error {
    sem := m.getSemaphore(providerID)
    if m.waitTimeout == 0 {
        select {
        case sem <- struct{}{}:
            return nil
        default:
            return &ConcurrencyError{Code: ErrConcurrencyLimitExceeded}
        }
    }
    // 带超时的等待
    waitCtx, cancel := context.WithTimeout(ctx, m.waitTimeout)
    defer cancel()
    select {
    case sem <- struct{}{}:
        return nil
    case <-waitCtx.Done():
        return &ConcurrencyError{Code: ErrConcurrencyLimitExceeded}
    }
}
```

### 3.2 RedisConcurrencyLimiter

基于 Redis 的分布式信号量，适用于多实例部署。

```go
// RedisConcurrencyLimiter 使用 Redis + Lua 脚本实现分布式信号量。
//
// Redis 数据结构：
//   key: concurrency:{providerID}
//   type: counter（INCR / DECR）
//
// Acquire Lua 脚本（原子操作）：
//   current = INCR key
//   if current > maxSlots:
//     DECR key
//     return 0  -- 失败
//   EXPIRE key 300  -- 防止进程崩溃后槽位泄漏
//   return 1  -- 成功
//
// Release Lua 脚本：
//   current = GET key
//   if current > 0: DECR key
type RedisConcurrencyLimiter struct {
    client      redis.Client
    maxSlots    map[string]int // providerID → 最大并发数
    waitTimeout time.Duration
    leaseExpiry time.Duration // 防槽位泄漏的 key 过期时间，默认 5min
}
```

<!-- @end-section -->

<!-- @section: config -->
---

## 4. 配置 Schema

```go
// ConcurrencyLimiterConfig 是 ConcurrencyLimiter 的配置。
type ConcurrencyLimiterConfig struct {
    // Backend 实现后端。
    Backend string `default:"memory" validate:"oneof=memory redis"`

    // RedisURL 仅 backend=redis 时需要。
    RedisURL string

    // WaitTimeout 等待获取并发槽位的最大时间。
    // 0 = 立即失败（不等待）；推荐生产设置 100ms~500ms。
    WaitTimeout time.Duration `default:"0"`

    // DefaultMax 未在 ProviderConfig 中配置时的默认最大并发数。
    DefaultMax int `default:"10" validate:"min=1,max=500"`

    // LeaseExpiry 仅 backend=redis：槽位的最大租约时间（防进程崩溃泄漏）。
    LeaseExpiry time.Duration `default:"5m"`
}
```

**各提供商的最大并发数**来自 `ProviderConfig.ConcurrencyMax`（由 `ProviderRegistry` 初始化时传入）：

```yaml
providers:
  - id: anthropic-primary
    concurrency_max: 20   # 最多 20 个并发请求

  - id: deepseek-primary
    concurrency_max: 50
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 5. 行为契约

| 契约 | 说明 |
|------|------|
| **Acquire/Release 配对** | 每次成功 Acquire 必须对应一次 Release（框架用 defer 保证） |
| **多余 Release 安全** | Release 多于 Acquire 时无操作（不 panic，不增加可用槽） |
| **WaitTimeout=0 立即失败** | 无可用槽时不等待，立即返回 ErrConcurrencyLimitExceeded |
| **ctx 取消时 Acquire 失败** | 等待期间 ctx 被取消，Acquire 返回 ctx.Err() |
| **Redis 实现防泄漏** | 进程崩溃后 LeaseExpiry 内槽位自动释放 |
| **Stats 不阻塞** | Stats() 是轻量快照查询，不得持有写锁 |

<!-- @end-section -->

<!-- @section: error-types -->
---

## 6. 错误类型

```go
type ConcurrencyError struct {
    Code    ConcurrencyErrorCode
    Message string
}

type ConcurrencyErrorCode string

const (
    // ErrConcurrencyLimitExceeded 并发槽已满，无法获取（超时或立即模式）。
    // 调用方返回 429（Too Many Requests）。
    ErrConcurrencyLimitExceeded ConcurrencyErrorCode = "CONCURRENCY_LIMIT_EXCEEDED"

    // ErrLimiterUnavailable 限流器后端不可用（Redis 故障等）。
    // 调用方可选择降级（允许请求）或拒绝。
    ErrLimiterUnavailable ConcurrencyErrorCode = "LIMITER_UNAVAILABLE"
)
```

<!-- @end-section -->

<!-- @section: testing -->
---

## 7. 测试支持

```go
// RunConcurrencyLimiterContractTests 验证 ConcurrencyLimiter 实现的行为契约。
func RunConcurrencyLimiterContractTests(t *testing.T, limiter ConcurrencyLimiter) {
    t.Run("Acquire/UnderLimit/Succeeds", ...)
    t.Run("Acquire/AtLimit/Blocks", ...)
    t.Run("Acquire/OverLimit/FailsImmediate", ...)           // WaitTimeout=0
    t.Run("Acquire/ContextCanceled/Returns", ...)
    t.Run("Release/ExtraRelease/Safe", ...)
    t.Run("ConcurrencySafety/ParallelAcquire/MaxRespected", ...)
}
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 8. 实现检查清单

```
ConcurrencyLimiter
  ☐ Acquire：WaitTimeout=0 时立即失败（不等待）
  ☐ Acquire：WaitTimeout>0 时等待至超时
  ☐ Acquire：ctx 取消时返回 ctx.Err()
  ☐ Release：多余调用安全（无操作）
  ☐ 最大并发数来自 ProviderConfig.ConcurrencyMax

MemoryConcurrencyLimiter
  ☐ 使用 chan struct{} 信号量
  ☐ 并发安全（无数据竞争）

RedisConcurrencyLimiter
  ☐ Acquire / Release 使用 Lua 脚本保证原子性
  ☐ LeaseExpiry 防止槽位泄漏
  ☐ Redis 不可用时返回 ErrLimiterUnavailable

测试
  ☐ 运行 RunConcurrencyLimiterContractTests（全部通过）
  ☐ 并发获取不超额（竞态测试，goroutine 数 >> MaxSlots）
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C06 在依赖图中的位置）
- model-provider-spec.md（C01，ProviderConfig.ConcurrencyMax 来源）
- rate-limiter-spec.md（C12，可选使用 ConcurrencyLimiter）

<!-- @end-section -->
