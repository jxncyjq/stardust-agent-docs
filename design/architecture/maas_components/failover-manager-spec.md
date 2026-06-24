---
id: "spec-component-failover-manager-040"
title: "FailoverManager 组件规范"
aliases: ["FailoverManager规范", "故障转移", "failover-manager-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "reliability", "failover", "retry", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C40"
layer: "L5"
depends_on:
  - "C41"   # RetryScheduler — 计算退避时间
optional_deps:
  - "C05"   # ProviderHealthMonitor — 缺失时不更新健康分，仅被动切换
conflicts_with: []
required_by: []
assembly_profiles:
  - standard
  - enterprise
  - embedded
---

<!-- @section: overview -->
# FailoverManager 组件规范

## 1. 组件定位

`FailoverManager` 负责单次请求内的**故障转移决策**：当某个提供商调用失败时，决定是重试当前提供商、切换到另一个提供商，还是终止请求。

它是请求管道中 `ProviderExecutor` 节点的**决策引擎**，与业务逻辑完全解耦。

```
ProviderExecutor
  │
  ├── 调用 ModelProvider ──► 成功 → 返回
  │                     └── 失败 ↓
  │
  └── FailoverManager.Decide(err)
          │
          ├── RetryCurrentProvider → 等待 RetryScheduler.Delay() → 重试
          ├── SwitchProvider       → 从 ProviderRegistry 选下一个
          └── Abort                → 返回错误给调用方
```

**读者**：实现可靠性策略的工程师、理解重试逻辑的管道开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

### 2.1 FailoverManager 接口

```go
// FailoverManager 管理单次请求内的故障转移状态和决策。
// 每次请求由 FailoverManagerFactory 创建一个新实例，实例不得跨请求复用。
// 非并发安全（单请求单实例）。
type FailoverManager interface {
    // Decide 根据错误信息和当前状态返回下一步决策。
    // providerID 是刚刚失败的提供商 ID。
    // 调用方必须严格按照 Decision 执行，不得忽略。
    Decide(providerID string, err error) Decision

    // RecordSuccess 记录成功调用（用于更新健康监控，非阻塞）。
    RecordSuccess(providerID string, latency time.Duration)

    // ExcludedProviders 返回当前轮次已排除（临时禁用）的提供商 ID 列表。
    // 由 ModelRouter 用于重新路由时过滤候选列表。
    ExcludedProviders() []string

    // Summary 返回本次请求的故障转移摘要（用于日志和遥测）。
    Summary() FailoverSummary
}

// FailoverManagerFactory 创建 FailoverManager 实例。
type FailoverManagerFactory interface {
    New(cfg FailoverConfig) FailoverManager
}

// Decision 是 Decide() 的返回值，描述下一步动作。
type Decision struct {
    Action      FailoverAction
    // RetryDelay 仅在 Action == RetryCurrentProvider 时有值。
    // 由 RetryScheduler 计算，调用方等待此时间后重试。
    RetryDelay  time.Duration
    // Reason 是人类可读的决策原因，用于日志。
    Reason      string
}

type FailoverAction int

const (
    // RetryCurrentProvider 立即或等待后重试同一提供商。
    RetryCurrentProvider FailoverAction = iota
    // SwitchProvider 切换到其他提供商（调用方需重新路由）。
    SwitchProvider
    // Abort 终止请求，返回错误给调用方。
    Abort
)
```

### 2.2 RetryScheduler 接口（C41）

```go
// RetryScheduler 计算重试等待时间，解耦退避算法。
type RetryScheduler interface {
    // Delay 返回第 attempt 次重试前的等待时间（1-indexed）。
    // 实现方可以是固定间隔、指数退避、抖动退避等。
    Delay(attempt int, errClass ErrorClass) time.Duration

    // MaxAttempts 返回针对该错误类型允许的最大重试次数。
    MaxAttempts(errClass ErrorClass) int
}
```

**内置 RetryScheduler 实现**：

| 实现 | 算法 | 适用场景 |
|------|------|----------|
| `FixedRetryScheduler` | 固定间隔 | 开发/测试 |
| `ExponentialBackoffScheduler` | 指数退避 + 抖动 | 生产默认 |
| `ProviderHintScheduler` | 遵从 `Retry-After` 响应头 | 限流类错误 |

<!-- @end-section -->

<!-- @section: decision-logic -->
---

## 3. 决策逻辑

### 3.1 错误分类到决策的映射

`FailoverManager` 内部先通过 `ModelProvider.ClassifyHTTPError()` 获取 `ErrorClass`，再映射到决策：

| ErrorClass | 默认决策 | 条件 |
|------------|----------|------|
| `Transient` | `RetryCurrentProvider` | 重试次数 < `MaxSameProviderRetry` |
| `Transient` | `SwitchProvider` | 重试次数 >= 上限 |
| `RateLimit` | `SwitchProvider` | 直接切换（不浪费重试次数） |
| `QuotaExhausted` | `SwitchProvider` + 临时禁用 | 禁用该提供商直到下次请求 |
| `BadRequest` | `Abort` | 重试无意义 |
| `AuthFailure` | `Abort` + 告警 | 触发 ProviderHealthMonitor.ReportAuthFailure() |
| `Unavailable` | `SwitchProvider` + 临时禁用 | 禁用并触发熔断计数 |
| `ContentFiltered` | `Abort` | 不重试，返回明确的 403 |

### 3.2 切换次数限制

```
单次请求内：
  同一提供商最大重试次数（SameProviderRetryMax）：默认 2
  总切换次数（MaxProviderSwitches）：默认 3

超过总切换次数 → Abort
没有更多可用提供商 → Abort（ExcludedProviders 排除后候选为空）
```

### 3.3 状态流转示意

```
请求开始
  │
  ▼
调用 Provider A ─── 失败 (Transient) ──► Decide → RetryCurrentProvider
                                                         │ delay=500ms
                                                         ▼
                          重试 Provider A ─── 失败 (Transient) ──► Decide
                                                         │ retry≥上限
                                                         ▼ SwitchProvider
                                                  (排除 Provider A)
                                                         │
                                                         ▼
                          调用 Provider B ─── 失败 (RateLimit) ──► Decide
                                                         │ SwitchProvider
                                                         ▼
                                                  (排除 Provider B)
                                                         │
                                                         ▼
                          调用 Provider C ──► 成功 → RecordSuccess → 返回
```

<!-- @end-section -->

<!-- @section: config -->
---

## 4. 配置 Schema

```go
// FailoverConfig 是 FailoverManager 的配置。
type FailoverConfig struct {
    // SameProviderRetryMax 同一提供商的最大重试次数（不含首次调用）。
    SameProviderRetryMax int `default:"2" validate:"min=0,max=10"`

    // MaxProviderSwitches 单次请求内最多切换提供商次数。
    MaxProviderSwitches int `default:"3" validate:"min=0,max=10"`

    // SwitchBackoffBase 每次切换提供商后的递增退避基数。
    // 第 n 次切换等待 n × SwitchBackoffBase（0 = 立即切换）。
    SwitchBackoffBase time.Duration `default:"0"`

    // TempDisableDuration 失败提供商的临时禁用时长。
    // 在当次请求内生效（通过 ExcludedProviders 实现），不影响全局健康状态。
    TempDisableDuration time.Duration `default:"0"` // 0 = 仅本次请求内禁用

    // ReportToHealthMonitor 是否将成功/失败上报给 ProviderHealthMonitor。
    // 需要 C05 已注册，否则此配置无效。
    ReportToHealthMonitor bool `default:"true"`
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 5. 行为契约

| 契约 | 说明 |
|------|------|
| **决策幂等** | 相同 `(providerID, errClass, retryCount)` 必须返回相同 Decision |
| **不做 IO** | `Decide()` 是纯内存操作，不得访问 DB 或 Redis |
| **ExcludedProviders 单调增** | 一旦排除的提供商不会在本次请求中恢复 |
| **RecordSuccess 非阻塞** | 必须立即返回，可异步上报给 HealthMonitor |
| **Abort 不重试** | 调用方收到 Abort 后必须立即终止，不得自行重试 |
| **切换后重新路由** | SwitchProvider 后，调用方必须通过 ModelRouter 重新选择，不得硬编码下一提供商 |

<!-- @end-section -->

<!-- @section: error-types -->
---

## 6. 错误类型与集成

### 6.1 FailoverManager 自身不返回 error

`Decide()` 总是返回一个有效的 `Decision`，不返回 error。
如果无法做出决策（极端情况），返回 `Decision{Action: Abort}`。

### 6.2 集成点：ProviderExecutor 伪代码

```go
func (e *ProviderExecutor) Execute(ctx *RequestContext) (*ModelOutput, error) {
    fm := e.fmFactory.New(e.cfg.Failover)

    for {
        provider, err := e.router.Route(ctx, fm.ExcludedProviders())
        if err != nil {
            return nil, ErrNoAvailableProvider
        }

        output, err := provider.ExecuteRequest(ctx, req)
        if err == nil {
            fm.RecordSuccess(provider.ID(), output.Latency)
            return output, nil
        }

        decision := fm.Decide(provider.ID(), err)
        switch decision.Action {
        case RetryCurrentProvider:
            time.Sleep(decision.RetryDelay)
            // 不更换 provider，继续循环
        case SwitchProvider:
            // ExcludedProviders 已更新，下次循环会排除失败的提供商
            time.Sleep(decision.RetryDelay)
        case Abort:
            return nil, err
        }
    }
}
```

<!-- @end-section -->

<!-- @section: testing -->
---

## 7. 测试支持

### 7.1 契约测试

```go
// RunFailoverManagerContractTests 验证决策逻辑和状态机。
func RunFailoverManagerContractTests(t *testing.T, factory FailoverManagerFactory) {
    t.Run("Transient/RetriesSameProviderBeforeSwitch", ...)
    t.Run("RateLimit/ImmediatelySwitches", ...)
    t.Run("AuthFailure/Aborts", ...)
    t.Run("ExcludedProviders/MonotonicallyGrows", ...)
    t.Run("MaxSwitches/AbortsWhenExceeded", ...)
    t.Run("RecordSuccess/NonBlocking", ...)
    t.Run("Decide/PureFunction", ...)  // 相同入参相同输出
}
```

### 7.2 场景测试矩阵

| 场景 | Provider A | Provider B | Provider C | 期望结果 |
|------|-----------|-----------|-----------|----------|
| 单次成功 | ✅ | — | — | 返回结果 |
| 重试后成功 | ❌ Transient → ✅ | — | — | 返回结果（1次重试） |
| 切换后成功 | ❌❌ Transient | ✅ | — | 返回结果（切换1次） |
| 全部失败 | ❌ | ❌ | ❌ | Abort |
| 限流 + 切换 | ❌ RateLimit | ✅ | — | 返回结果（立即切换） |
| 认证失败 | ❌ Auth | — | — | Abort + 告警 |

<!-- @end-section -->

<!-- @section: checklist -->
---

## 8. 实现检查清单

```
FailoverManager
  ☐ Decide：根据 ErrorClass 和重试计数正确映射 FailoverAction
  ☐ Decide：纯内存操作，无 IO
  ☐ ExcludedProviders：失败提供商立即加入排除列表
  ☐ MaxProviderSwitches：超限后返回 Abort
  ☐ RecordSuccess：异步上报，不阻塞

RetryScheduler（C41）
  ☐ Delay：返回值 >= 0
  ☐ 指数退避实现包含抖动（避免惊群）
  ☐ MaxAttempts：针对不同 ErrorClass 返回合理上限

测试
  ☐ 运行 RunFailoverManagerContractTests（全部通过）
  ☐ 覆盖场景测试矩阵中所有行
  ☐ Decide 纯函数验证（相同入参相同输出）
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C40 在依赖图中的位置）
- retry-scheduler-spec.md（C41，必须依赖）
- provider-health-monitor-spec.md（C05，可选依赖）
- [[model-provider-spec|ModelProvider 组件规范]]（C01，提供 ClassifyHTTPError）

<!-- @end-section -->
