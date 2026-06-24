---
id: "spec-component-circuit-breaker-034"
title: "CircuitBreaker 组件规范"
aliases: ["CircuitBreaker规范", "熔断器", "circuit-breaker-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "reliability", "circuit-breaker", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C34"
layer: "L4"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "C33"   # QuotaBudgetManager 可选使用 CircuitBreaker 自动熔断超额租户
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# CircuitBreaker 组件规范

## 1. 组件定位

`CircuitBreaker` 是**通用熔断器**，供 `QuotaBudgetManager` 等组件使用，当某个维度（租户/用户组）的错误率或拒绝率持续超阈值时自动熔断，避免持续压测数据库。

> **注意**：`ProviderHealthMonitor`（C05）内部已内置针对提供商的熔断逻辑。本组件是一个独立的、可复用的熔断原语，供其他组件按需引用。

```
QuotaBudgetManager.Consume() 失败
        │
        ▼
CircuitBreaker.RecordFailure(key)
        │
        ▼ 若失败率 > 阈值
CircuitBreaker.State(key) → OPEN
        │
        ▼ 后续 Consume 在检查前调用 CircuitBreaker.Allow(key)
        └── OPEN → 直接拒绝（不查 DB）
```

**读者**：需要自动熔断保护的组件开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// CircuitBreaker 通用熔断器原语。
// 并发安全：所有方法可被多个 goroutine 并发调用。
type CircuitBreaker interface {
    // Allow 检查指定 key 是否允许执行操作。
    // CLOSED/HALF_OPEN(探测)→ true；OPEN → false。
    // HALF_OPEN 状态下每次仅允许一个探测请求通过（框架保证）。
    Allow(key string) bool

    // RecordSuccess 记录一次成功（用于从 OPEN/HALF_OPEN 恢复）。
    RecordSuccess(key string)

    // RecordFailure 记录一次失败（用于触发熔断）。
    RecordFailure(key string)

    // State 返回当前状态（用于监控）。
    State(key string) CircuitState

    // Reset 手动重置熔断器（管理员操作）。
    Reset(key string)
}

// CircuitState 熔断器状态。
type CircuitState int

const (
    CircuitClosed   CircuitState = iota // 正常
    CircuitOpen                         // 熔断
    CircuitHalfOpen                     // 半开（探测）
)
```

<!-- @end-section -->

<!-- @section: config -->
---

## 3. 配置 Schema

```go
// CircuitBreakerConfig 熔断器配置。
type CircuitBreakerConfig struct {
    // FailureThreshold 触发熔断的连续失败次数。
    FailureThreshold int `default:"5"`

    // FailureRateThreshold 触发熔断的错误率（需 MinRequests 满足）。
    FailureRateThreshold float64 `default:"0.6" validate:"min=0,max=1"`

    // MinRequests 计算错误率所需的最小请求数。
    MinRequests int `default:"10"`

    // ResetTimeout 熔断后等待多久进入 HALF_OPEN。
    ResetTimeout time.Duration `default:"30s"`

    // WindowSize 滑动窗口大小（请求数）。
    WindowSize int `default:"100"`
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 4. 行为契约

| 契约 | 说明 |
|------|------|
| **Allow 幂等** | CLOSED 状态多次调用均返回 true |
| **HALF_OPEN 单探测** | 同一时刻只允许一个请求探测（其余仍返回 false）|
| **key 隔离** | 不同 key 的熔断状态完全独立 |
| **Noop 实现始终 Allow** | C34 未注册时框架注入 Noop（永不熔断）|

<!-- @end-section -->

<!-- @section: checklist -->
---

## 5. 实现检查清单

```
CircuitBreaker
  ☐ 三态状态机：CLOSED → OPEN → HALF_OPEN → CLOSED
  ☐ HALF_OPEN：同时只允许一个探测请求
  ☐ key 维度完全隔离
  ☐ 并发安全（无数据竞争）

Noop 实现
  ☐ Allow 始终返回 true
  ☐ Record* 和 Reset 为无操作

测试
  ☐ 状态转换完整覆盖
  ☐ HALF_OPEN 单探测并发验证
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C34 在依赖图中的位置）
- quota-budget-manager-spec.md（C33，主要消费者）
- provider-health-monitor-spec.md（C05，内置针对提供商的熔断逻辑，概念参考）

<!-- @end-section -->
