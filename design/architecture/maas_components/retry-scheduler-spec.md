---
id: "spec-component-retry-scheduler-041"
title: "RetryScheduler 组件规范"
aliases: ["RetryScheduler规范", "重试调度器", "retry-scheduler-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "reliability", "retry", "backoff", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C41"
layer: "L5"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "C40"   # FailoverManager 使用 RetryScheduler 计算退避时间
assembly_profiles:
  - standard
  - enterprise
  - embedded
---

<!-- @section: overview -->
# RetryScheduler 组件规范

## 1. 组件定位

`RetryScheduler` 是可靠性层的**退避算法引擎**，负责计算重试等待时间和最大重试次数。它将退避策略从 `FailoverManager` 的决策逻辑中解耦，使两者可以独立变化。

```
FailoverManager
        │
        │ Decide(err) → RetryCurrentProvider
        │
        ▼
  RetryScheduler
  ┌──────────────────────────────────────────┐
  │  Delay(attempt, errClass)                │ ← 返回等待时间
  │  MaxAttempts(errClass)                   │ ← 返回最大重试次数
  │                                          │
  │  内置三种算法：                           │
  │    FixedRetryScheduler      → 固定间隔   │
  │    ExponentialBackoffScheduler → 指数退避 │
  │    ProviderHintScheduler    → 遵从响应头  │
  └──────────────────────────────────────────┘
```

**读者**：配置重试策略的运维工程师、实现自定义退避算法的开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// RetryScheduler 计算重试等待时间，解耦退避算法与决策逻辑。
// 并发安全：同一实例可在多个 goroutine 中并发调用（无可变状态）。
type RetryScheduler interface {
    // Delay 返回第 attempt 次重试前的等待时间（attempt 从 1 开始）。
    // errClass 用于针对不同错误类型调整退避策略。
    // 返回值 >= 0；返回 0 表示立即重试（不等待）。
    Delay(attempt int, errClass ErrorClass) time.Duration

    // MaxAttempts 返回针对该错误类型允许的最大重试次数（不含首次调用）。
    // 返回 0 表示不重试（首次失败即转为 SwitchProvider 或 Abort）。
    MaxAttempts(errClass ErrorClass) int
}
```

**注意**：`RetryScheduler` 只负责**计算**，不负责实际等待（`time.Sleep`）。等待由 `FailoverManager` 的调用方（`ProviderExecutor`）执行。

<!-- @end-section -->

<!-- @section: implementations -->
---

## 3. 内置实现

### 3.1 FixedRetryScheduler

固定间隔，适用于开发/测试环境。

```go
// FixedRetryScheduler 对所有重试返回相同的固定等待时间。
type FixedRetryScheduler struct {
    Interval    time.Duration         // 固定等待间隔，默认 500ms
    MaxRetries  map[ErrorClass]int    // 各错误类型的最大重试次数
}

func (f *FixedRetryScheduler) Delay(attempt int, _ ErrorClass) time.Duration {
    return f.Interval
}

func (f *FixedRetryScheduler) MaxAttempts(errClass ErrorClass) int {
    if n, ok := f.MaxRetries[errClass]; ok {
        return n
    }
    return 2 // 默认重试 2 次
}
```

**默认配置**：

```go
var DefaultFixedScheduler = &FixedRetryScheduler{
    Interval: 500 * time.Millisecond,
    MaxRetries: map[ErrorClass]int{
        ErrClassTransient: 2,
        ErrClassRateLimit: 0, // 不重试，直接切换
        ErrClassUnavailable: 0,
    },
}
```

### 3.2 ExponentialBackoffScheduler（生产默认）

指数退避 + 抖动，避免惊群效应（Thundering Herd）。

**公式**：

```
base_delay = BaseDelay × Multiplier^(attempt-1)
capped_delay = min(base_delay, MaxDelay)
jitter = capped_delay × JitterFactor × rand[0, 1)
final_delay = capped_delay + jitter
```

```go
// ExponentialBackoffScheduler 实现带抖动的指数退避。
type ExponentialBackoffScheduler struct {
    BaseDelay    time.Duration         // 首次重试等待，默认 200ms
    Multiplier   float64               // 退避倍数，默认 2.0
    MaxDelay     time.Duration         // 最大等待上限，默认 30s
    JitterFactor float64               // 抖动系数 [0, 1]，默认 0.3
    MaxRetries   map[ErrorClass]int    // 各错误类型的最大重试次数
}

func (e *ExponentialBackoffScheduler) Delay(attempt int, errClass ErrorClass) time.Duration {
    base := float64(e.BaseDelay) * math.Pow(e.Multiplier, float64(attempt-1))
    capped := math.Min(base, float64(e.MaxDelay))
    jitter := capped * e.JitterFactor * rand.Float64()
    return time.Duration(capped + jitter)
}
```

**默认配置**：

```go
var DefaultExponentialScheduler = &ExponentialBackoffScheduler{
    BaseDelay:    200 * time.Millisecond,
    Multiplier:   2.0,
    MaxDelay:     30 * time.Second,
    JitterFactor: 0.3,
    MaxRetries: map[ErrorClass]int{
        ErrClassTransient:   2,
        ErrClassRateLimit:   0, // 不重试当前提供商，切换
        ErrClassUnavailable: 0,
    },
}
```

**重试时间序列示例**（BaseDelay=200ms, Multiplier=2, JitterFactor=0.3）：

| Attempt | Base | Cap | Jitter范围 | 实际范围 |
|---------|------|-----|-----------|---------|
| 1 | 200ms | 200ms | 0~60ms | 200~260ms |
| 2 | 400ms | 400ms | 0~120ms | 400~520ms |
| 3 | 800ms | 800ms | 0~240ms | 800~1040ms |
| 4 | 1.6s | 1.6s | 0~480ms | 1.6~2.08s |

### 3.3 ProviderHintScheduler

遵从提供商响应头中的 `Retry-After`，适用于限流错误。

```go
// ProviderHintScheduler 从错误上下文中读取提供商建议的等待时间。
// 仅对 ErrClassRateLimit 生效；其他错误类型委托给内置的 fallback。
type ProviderHintScheduler struct {
    // MaxHintDelay 接受的最大提供商建议等待时间。
    // 超过此值则忽略 Retry-After，直接切换提供商。
    MaxHintDelay time.Duration // 默认 60s
    Fallback     RetryScheduler
}

// Delay 从 errClass 附带的 ProviderError.RetryAfter 中读取等待时间。
// 若 RetryAfter 为 0 或超过 MaxHintDelay，委托给 Fallback。
func (p *ProviderHintScheduler) Delay(attempt int, errClass ErrorClass) time.Duration {
    // RetryAfter 通过 context.Value 从 ProviderError 传递（框架注入）
    // 此处简化展示核心逻辑
    if retryAfter := getRetryAfterFromContext(); retryAfter > 0 && retryAfter <= p.MaxHintDelay {
        return retryAfter
    }
    return p.Fallback.Delay(attempt, errClass)
}

func (p *ProviderHintScheduler) MaxAttempts(errClass ErrorClass) int {
    if errClass == ErrClassRateLimit {
        return 1 // 遵从提示后最多重试 1 次
    }
    return p.Fallback.MaxAttempts(errClass)
}
```

<!-- @end-section -->

<!-- @section: error-class-mapping -->
---

## 4. ErrorClass 与重试策略映射

各内置实现对 `ErrorClass` 的默认策略：

| ErrorClass | MaxAttempts | 说明 |
|------------|-------------|------|
| `Transient` | 2 | 临时错误可重试 |
| `RateLimit` | 0 | 不重试当前提供商，由 FailoverManager 切换 |
| `QuotaExhausted` | 0 | 配额耗尽，切换提供商 |
| `BadRequest` | 0 | 请求有误，不重试 |
| `AuthFailure` | 0 | 认证失败，不重试 |
| `Unavailable` | 0 | 提供商不可用，切换 |
| `ContentFiltered` | 0 | 内容被过滤，终止 |

> **说明**：`MaxAttempts=0` 表示不重试当前提供商，`FailoverManager` 会根据 ErrorClass 决定是切换还是终止（Abort）。`RetryScheduler` 只参与"重试当前提供商"的决策。

<!-- @end-section -->

<!-- @section: config -->
---

## 5. 配置 Schema

```go
// RetrySchedulerConfig 通过 YAML 配置文件配置。
// 框架根据 Type 字段实例化对应的实现。
type RetrySchedulerConfig struct {
    // Type 调度器类型。
    Type string `default:"exponential" validate:"oneof=fixed exponential provider_hint"`

    // Fixed 仅 type=fixed 时有效。
    Fixed *FixedSchedulerConfig

    // Exponential 仅 type=exponential 时有效。
    Exponential *ExponentialSchedulerConfig

    // ProviderHint 仅 type=provider_hint 时有效（含内嵌 fallback 配置）。
    ProviderHint *ProviderHintSchedulerConfig
}

type ExponentialSchedulerConfig struct {
    BaseDelay    time.Duration `default:"200ms" validate:"min=10ms,max=10s"`
    Multiplier   float64       `default:"2.0"   validate:"min=1.1,max=10"`
    MaxDelay     time.Duration `default:"30s"   validate:"min=1s,max=300s"`
    JitterFactor float64       `default:"0.3"   validate:"min=0,max=1"`
    MaxRetries   map[string]int // ErrorClass 名称 → 最大重试次数
}
```

**YAML 示例**：

```yaml
retry_scheduler:
  type: exponential
  exponential:
    base_delay: 200ms
    multiplier: 2.0
    max_delay: 30s
    jitter_factor: 0.3
    max_retries:
      Transient: 2
      RateLimit: 0
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 6. 行为契约

| 契约 | 说明 |
|------|------|
| **Delay 返回值 >= 0** | 不得返回负值；0 表示立即重试 |
| **MaxAttempts 返回值 >= 0** | 0 表示不重试（不含首次调用） |
| **纯函数（无副作用）** | `Delay()` 和 `MaxAttempts()` 不得修改任何状态 |
| **并发安全** | 多 goroutine 并发调用必须安全（无共享可变状态） |
| **指数退避必须包含抖动** | 避免多个 goroutine 同时重试造成惊群效应 |
| **不做实际等待** | 调度器只计算时间，`time.Sleep` 由调用方执行 |
| **MaxDelay 是硬上限** | 无论 attempt 多大，Delay 不得超过 MaxDelay |

<!-- @end-section -->

<!-- @section: testing -->
---

## 7. 测试支持

### 7.1 契约测试

```go
// RunRetrySchedulerContractTests 验证 RetryScheduler 实现的行为契约。
func RunRetrySchedulerContractTests(t *testing.T, scheduler RetryScheduler) {
    t.Run("Delay/NonNegative", ...)
    t.Run("Delay/NotExceedMaxDelay", ...)
    t.Run("MaxAttempts/NonNegative", ...)
    t.Run("ExponentialGrowth/DelayIncreasesWithAttempt", ...)
    t.Run("Jitter/SameInputDifferentOutputs", ...)      // 验证有抖动
    t.Run("ConcurrencySafety/ParallelDelay", ...)
}
```

### 7.2 确定性测试（固定随机种子）

```go
// 指数退避测试：验证抖动范围，使用固定 rand source 保证可重复
func TestExponentialBackoff_JitterRange(t *testing.T) {
    scheduler := &ExponentialBackoffScheduler{
        BaseDelay: 200 * time.Millisecond,
        Multiplier: 2.0,
        MaxDelay: 30 * time.Second,
        JitterFactor: 0.3,
    }

    for attempt := 1; attempt <= 5; attempt++ {
        delays := make([]time.Duration, 100)
        for i := range delays {
            delays[i] = scheduler.Delay(attempt, ErrClassTransient)
        }

        base := time.Duration(float64(200*time.Millisecond) * math.Pow(2, float64(attempt-1)))
        capped := min(base, 30*time.Second)
        minExpected := capped
        maxExpected := capped + time.Duration(float64(capped)*0.3)

        for _, d := range delays {
            assert.GreaterOrEqual(t, d, minExpected)
            assert.LessOrEqual(t, d, maxExpected)
        }
    }
}
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 8. 实现检查清单

```
RetryScheduler（通用）
  ☐ Delay：返回值 >= 0，不超过 MaxDelay
  ☐ MaxAttempts：返回值 >= 0
  ☐ 纯函数：无副作用，无可变状态
  ☐ 并发安全

ExponentialBackoffScheduler
  ☐ 抖动：相同入参产生不同输出（rand 参与）
  ☐ 增长：attempt 增大时 Delay 单调不减（基准值）
  ☐ 上限：Delay <= MaxDelay

ProviderHintScheduler
  ☐ 读取 ProviderError.RetryAfter（通过 context 或参数传递）
  ☐ RetryAfter > MaxHintDelay 时委托 Fallback
  ☐ RetryAfter == 0 时委托 Fallback

测试
  ☐ 运行 RunRetrySchedulerContractTests（全部通过）
  ☐ 抖动范围验证（100次采样在 [base, base+jitter_max] 内）
  ☐ MaxDelay 边界测试（attempt 足够大时不超上限）
  ☐ 并发安全验证
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C41 在依赖图中的位置）
- failover-manager-spec.md（C40，主要消费者）
- [[model-provider-spec|ModelProvider 组件规范]]（C01，提供 ErrorClass 定义）

<!-- @end-section -->
