---
id: "spec-component-provider-health-monitor-005"
title: "ProviderHealthMonitor 组件规范"
aliases: ["ProviderHealthMonitor规范", "健康监控", "provider-health-monitor-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "reliability", "health", "circuit-breaker", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C05"
layer: "L1"
depends_on:
  - "C01"   # ModelProvider — 监控对象，可选调用 ProviderWithHealthCheck
optional_deps: []
conflicts_with: []
required_by:
  - "C14"   # ModelRouter 读取健康分用于路由过滤
  - "C23"   # HealthAwareStrategy 读取健康分用于路由排序
  - "C40"   # FailoverManager 上报成功/失败事件
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# ProviderHealthMonitor 组件规范

## 1. 组件定位

`ProviderHealthMonitor` 持续追踪每个已注册提供商的**健康状态**，为路由层提供实时健康分，并维护熔断状态。

它同时支持两种探测方式：
- **主动探测**：定期调用 `ProviderWithHealthCheck.HealthCheck()`（如提供商实现了该接口）
- **被动上报**：由 `FailoverManager` 在请求失败/成功时调用 `ReportFailure / ReportSuccess`

```
                    主动探测 (周期性)
ProviderHealthMonitor ←── HealthCheck() ──► ModelProvider
        │
        │ 被动上报
        ├── ReportFailure(providerID, errClass)  ← FailoverManager 调用
        ├── ReportSuccess(providerID, latency)   ← FailoverManager 调用
        │
        │ 健康分查询
        ├── HealthScore(providerID) → float64    ← ModelRouter / HealthAwareStrategy
        └── IsCircuitOpen(providerID) → bool     ← ModelRouter 过滤熔断提供商
```

**读者**：配置健康检查策略的运维工程师、集成故障转移的开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

### 2.1 ProviderHealthMonitor 接口

```go
// ProviderHealthMonitor 追踪提供商健康状态并提供路由所需的健康分。
// 并发安全：所有方法可被多个 goroutine 并发调用。
type ProviderHealthMonitor interface {
    // HealthScore 返回指定提供商的当前健康分（0.0 = 熔断，1.0 = 完全健康）。
    // providerID 未注册时返回 1.0（宽松策略，不影响新提供商的初始路由）。
    HealthScore(providerID string) float64

    // IsCircuitOpen 返回提供商的熔断器是否处于断开状态（true = 熔断，禁止路由）。
    // 熔断状态下 HealthScore 为 0.0，但调用方可直接用此方法避免浮点比较。
    IsCircuitOpen(providerID string) bool

    // ReportFailure 上报一次失败事件（由 FailoverManager 异步调用）。
    // errClass 用于区分错误类型：AuthFailure 触发告警，Transient 仅计入错误率。
    ReportFailure(providerID string, errClass ErrorClass)

    // ReportSuccess 上报一次成功事件（由 FailoverManager 异步调用）。
    // latency 用于计算 P99 延迟指标（不影响健康分，仅用于遥测）。
    ReportSuccess(providerID string, latency time.Duration)

    // ReportAuthFailure 上报认证失败（特殊处理：触发告警并临时禁用提供商）。
    // 认证失败通常需要人工介入（API Key 过期），告警比自动恢复更重要。
    ReportAuthFailure(providerID string)

    // ForceDisable 手动禁用提供商（管理员操作，设置熔断状态）。
    // 直到 ForceEnable 调用前持续禁用。
    ForceDisable(providerID string, reason string)

    // ForceEnable 手动恢复提供商（清除强制禁用状态，恢复正常健康计算）。
    ForceEnable(providerID string)
}
```

### 2.2 健康分计算模型

```go
// HealthState 是每个提供商的内部健康状态。
type HealthState struct {
    ProviderID      string
    Score           float64        // 当前健康分 [0.0, 1.0]
    CircuitOpen     bool           // 熔断器状态
    ForceDisabled   bool           // 管理员强制禁用
    SuccessCount    int64          // 滑动窗口内成功次数
    FailureCount    int64          // 滑动窗口内失败次数
    ConsecFailures  int            // 连续失败次数（用于熔断触发）
    LastFailureAt   time.Time
    LastSuccessAt   time.Time
    LastCheckAt     time.Time      // 最后一次主动健康检查时间
    CircuitOpenAt   time.Time      // 熔断开启时间（用于半开状态计时）
}
```

<!-- @end-section -->

<!-- @section: health-score -->
---

## 3. 健康分计算

### 3.1 滑动窗口错误率

```
健康分 = 1.0 - ErrorRate(window)

ErrorRate = FailureCount / (SuccessCount + FailureCount)
```

- **窗口类型**：时间滑动窗口（默认 60 秒）或计数滑动窗口（默认最近 100 次请求）
- **最小样本数**：窗口内请求数 < MinSampleCount（默认 5）时，健康分固定为 1.0（避免冷启动惩罚）

### 3.2 熔断逻辑（Circuit Breaker）

```
熔断触发条件（任一）：
  1. 连续失败次数 >= ConsecFailureThreshold（默认 5）
  2. ErrorRate > CircuitOpenThreshold（默认 0.8）且 RequestCount >= MinSampleCount

熔断状态：
  CLOSED  → 正常路由
  OPEN    → 熔断，HealthScore = 0.0，不参与路由
  HALF_OPEN → 熔断后等待 ResetTimeout，允许一次探测请求
              → 探测成功 → CLOSED；探测失败 → OPEN（重置计时）
```

```go
// CircuitBreakerConfig 熔断器配置（嵌入 HealthMonitorConfig）。
type CircuitBreakerConfig struct {
    ConsecFailureThreshold int           `default:"5"`
    CircuitOpenThreshold   float64       `default:"0.8"`
    MinSampleCount         int           `default:"5"`
    ResetTimeout           time.Duration `default:"30s"`
    HalfOpenProbeTimeout   time.Duration `default:"5s"`
}
```

### 3.3 错误类型权重

不同错误类型对健康分的影响权重不同：

| ErrorClass | 权重 | 说明 |
|------------|------|------|
| `Transient` | 1.0 | 标准错误计入 |
| `Unavailable` | 1.0 | 标准错误计入 |
| `RateLimit` | 0.5 | 限流不代表提供商不健康，轻微扣分 |
| `QuotaExhausted` | 0.0 | 配额问题不影响健康分（配置问题） |
| `BadRequest` | 0.0 | 请求问题不影响提供商健康分 |
| `AuthFailure` | 0.0（分开处理）| 触发 `ReportAuthFailure`，特殊告警路径 |
| `ContentFiltered` | 0.0 | 内容策略不影响健康分 |

<!-- @end-section -->

<!-- @section: active-probe -->
---

## 4. 主动健康探测

```go
// 框架内部的探测调度器（不暴露为接口）
type ProbeScheduler struct {
    monitor  ProviderHealthMonitor
    interval time.Duration // 默认 30s
}

// 探测流程：
//   1. 遍历所有已注册提供商
//   2. 若提供商实现了 ProviderWithHealthCheck，调用 HealthCheck(ctx)
//   3. HealthCheck 成功 → ReportSuccess（latency=探测耗时）
//   4. HealthCheck 失败 → ReportFailure（errClass=Transient）
//   5. 未实现 ProviderWithHealthCheck → 跳过主动探测（仅依赖被动上报）
```

**探测配置**：

```go
type ActiveProbeConfig struct {
    Enabled  bool          `default:"true"`
    Interval time.Duration `default:"30s"`
    Timeout  time.Duration `default:"5s"`
    // ProbeOnlyUnhealthy 仅对健康分 < 阈值的提供商探测（减少对健康提供商的探测压力）
    ProbeOnlyUnhealthy bool    `default:"false"`
    UnhealthyThreshold float64 `default:"0.8"`
}
```

<!-- @end-section -->

<!-- @section: config -->
---

## 5. 配置 Schema

```go
// HealthMonitorConfig 是 ProviderHealthMonitor 的完整配置。
type HealthMonitorConfig struct {
    // Window 健康分计算的滑动窗口大小。
    WindowDuration time.Duration `default:"60s"`
    WindowSize     int           `default:"100"` // 计数窗口（二选一）

    // MinSampleCount 最小样本数，低于此值健康分固定为 1.0。
    MinSampleCount int `default:"5"`

    // CircuitBreaker 熔断器配置。
    CircuitBreaker CircuitBreakerConfig

    // ActiveProbe 主动探测配置。
    ActiveProbe ActiveProbeConfig

    // AuthFailureDisableDuration 认证失败后临时禁用时长（0 = 永久禁用直到 ForceEnable）。
    AuthFailureDisableDuration time.Duration `default:"0"`
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 6. 行为契约

| 契约 | 说明 |
|------|------|
| **HealthScore 始终在 [0.0, 1.0]** | 不得返回范围外的值 |
| **未注册提供商返回 1.0** | 新提供商首次被路由时不受惩罚 |
| **ReportFailure 非阻塞** | 必须立即返回，不等待健康分重新计算 |
| **ReportSuccess 非阻塞** | 同上 |
| **熔断开启后 HealthScore == 0.0** | 路由层可用 HealthScore == 0 或 IsCircuitOpen() 检测 |
| **AuthFailure 触发告警** | ReportAuthFailure 必须触发外部告警（日志 + 指标），不仅仅是更新状态 |
| **ForceDisable 优先于自动恢复** | 即使主动探测成功，ForceDisable 状态下也不恢复 |

<!-- @end-section -->

<!-- @section: testing -->
---

## 7. 测试支持

```go
// RunHealthMonitorContractTests 验证 ProviderHealthMonitor 实现的行为契约。
func RunHealthMonitorContractTests(t *testing.T, monitor ProviderHealthMonitor) {
    t.Run("HealthScore/UnknownProvider/Returns1", ...)
    t.Run("HealthScore/AfterFailures/Decreases", ...)
    t.Run("CircuitOpen/AfterConsecFailures/Score0", ...)
    t.Run("CircuitOpen/HalfOpen/AfterResetTimeout", ...)
    t.Run("ReportFailure/NonBlocking", ...)
    t.Run("ForceDisable/PreventsAutoRecovery", ...)
    t.Run("ConcurrencySafety/ParallelReport", ...)
}

// NoopHealthMonitor 用于测试和 minimal 装配方案（C05 未注册时的默认实现）。
type NoopHealthMonitor struct{}

func (n *NoopHealthMonitor) HealthScore(_ string) float64    { return 1.0 }
func (n *NoopHealthMonitor) IsCircuitOpen(_ string) bool     { return false }
func (n *NoopHealthMonitor) ReportFailure(_ string, _ ErrorClass) {}
func (n *NoopHealthMonitor) ReportSuccess(_ string, _ time.Duration) {}
func (n *NoopHealthMonitor) ReportAuthFailure(_ string)       {}
func (n *NoopHealthMonitor) ForceDisable(_ string, _ string)  {}
func (n *NoopHealthMonitor) ForceEnable(_ string)             {}
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 8. 实现检查清单

```
ProviderHealthMonitor
  ☐ HealthScore：范围 [0.0, 1.0]，未注册提供商返回 1.0
  ☐ IsCircuitOpen：熔断时返回 true，HealthScore 同时为 0.0
  ☐ ReportFailure / ReportSuccess：非阻塞，立即返回
  ☐ 滑动窗口：时间窗口或计数窗口（二选一）
  ☐ MinSampleCount：样本不足时健康分为 1.0

熔断器
  ☐ 连续失败 >= ConsecFailureThreshold → OPEN
  ☐ ErrorRate > CircuitOpenThreshold → OPEN
  ☐ OPEN 后等待 ResetTimeout → HALF_OPEN
  ☐ HALF_OPEN 探测成功 → CLOSED；失败 → OPEN
  ☐ 错误类型权重：RateLimit=0.5，AuthFailure分开处理

主动探测
  ☐ 实现了 ProviderWithHealthCheck 才发起探测
  ☐ 探测结果通过 ReportSuccess / ReportFailure 更新

认证失败
  ☐ ReportAuthFailure 触发外部告警（日志 ERROR + 指标）
  ☐ 支持 AuthFailureDisableDuration 配置

管理员控制
  ☐ ForceDisable 优先于所有自动逻辑
  ☐ ForceEnable 清除强制禁用，恢复正常计算

测试
  ☐ 运行 RunHealthMonitorContractTests（全部通过）
  ☐ 熔断状态转换完整覆盖（CLOSED → OPEN → HALF_OPEN → CLOSED）
  ☐ 并发安全：多 goroutine 同时上报
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C05 在依赖图中的位置）
- model-provider-spec.md（C01，ProviderWithHealthCheck 可选接口）
- model-router-spec.md（C14，读取健康分用于候选过滤）
- failover-manager-spec.md（C40，上报成功/失败事件）

<!-- @end-section -->
