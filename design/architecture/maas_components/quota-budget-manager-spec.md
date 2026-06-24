---
id: "spec-component-quota-budget-manager-033"
title: "QuotaBudgetManager 组件规范"
aliases: ["QuotaBudgetManager规范", "配额预算管理器", "quota-budget-manager-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "billing", "quota", "budget", "rate-limit", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C33"
layer: "L4"
depends_on: []
optional_deps:
  - "C34"   # CircuitBreaker — 缺失时不做自动熔断，仅拒绝超额请求
conflicts_with: []
required_by:
  - "C13"   # QuotaChecker 在请求入口做快速预检
  - "C31"   # BillingSession 在计费时做多级配额检查
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# QuotaBudgetManager 组件规范

## 1. 组件定位

`QuotaBudgetManager` 实现**多级配额预算管理**：按用户、用户组、租户、全局等不同维度设置配额上限，在请求入口（`QuotaChecker`）和计费阶段（`BillingSession`）分别执行检查。

```
请求入口
  QuotaChecker (C13)
        │ Check(ctx) — 快速预检，不扣款
        ▼
  QuotaBudgetManager
  ┌──────────────────────────────────────────────────────┐
  │  配额维度（从细到粗，任一超额即拒绝）：               │
  │    User   → group → tenant → global                  │
  │                                                      │
  │  Check(ctx) → 是否允许本次请求                       │
  │  Consume(ctx, units) → 扣减配额并记录                │
  │  Refund(ctx, requestID) → 退还预扣配额               │
  └──────────────────────────────────────────────────────┘
```

**读者**：配置配额策略的运营人员、实现计费集成的开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// QuotaBudgetManager 管理多级配额预算，支持按用户/组/租户/全局维度限额。
// 并发安全：所有方法可被多个 goroutine 并发调用。
type QuotaBudgetManager interface {
    // Check 检查当前请求是否在配额范围内（不扣减配额）。
    // 用于 QuotaChecker 的请求入口快速预检。
    // estimatedUnits 是预估消耗量（来自 PricingEngine.Estimate，可为 0 做初步检查）。
    // 返回 nil 表示允许，返回 *QuotaError 表示拒绝（含具体超额的维度）。
    Check(ctx *RequestContext, estimatedUnits int64) error

    // Consume 扣减配额（在 BillingSession.PreConsume 内调用）。
    // requestID 是幂等键，相同 requestID 重复调用不重复扣减。
    // 返回 *QuotaError 时 BillingSession 应回滚并返回 402。
    Consume(ctx *RequestContext, requestID string, units int64) error

    // Refund 退还已扣减的配额（在 BillingSession.Refund 内调用）。
    // 幂等：相同 requestID 多次调用只退还一次。
    Refund(ctx *RequestContext, requestID string) error

    // Adjust 结算差额（在 BillingSession.Settle 内调用）。
    // delta > 0：补扣；delta < 0：退还超扣部分。
    Adjust(ctx *RequestContext, requestID string, delta int64) error

    // BudgetStatus 返回当前配额状态快照（用于管理接口和遥测）。
    BudgetStatus(ctx *RequestContext) BudgetStatusSnapshot
}
```

### 2.2 相关类型

```go
// BudgetDimension 配额维度。
type BudgetDimension string

const (
    DimensionUser   BudgetDimension = "user"
    DimensionGroup  BudgetDimension = "group"
    DimensionTenant BudgetDimension = "tenant"
    DimensionGlobal BudgetDimension = "global"
)

// BudgetPeriod 配额周期。
type BudgetPeriod string

const (
    PeriodMinute BudgetPeriod = "minute"  // RPM 限制
    PeriodHour   BudgetPeriod = "hour"
    PeriodDay    BudgetPeriod = "day"
    PeriodMonth  BudgetPeriod = "month"   // 月度配额（最常用）
    PeriodTotal  BudgetPeriod = "total"   // 永久总量
)

// BudgetStatusSnapshot 配额状态快照。
type BudgetStatusSnapshot struct {
    Dimensions []DimensionStatus
}

type DimensionStatus struct {
    Dimension BudgetDimension
    Period    BudgetPeriod
    Limit     int64   // 配额上限（-1 表示无限）
    Used      int64   // 当前周期已用
    Remaining int64   // 剩余（Limit - Used，-1 表示无限）
    ResetAt   *time.Time // 下次重置时间（Total 类型为 nil）
}
```

<!-- @end-section -->

<!-- @section: budget-rules -->
---

## 3. 配额规则

### 3.1 多级检查顺序

```
Check 按以下顺序检查（从细粒度到粗粒度），任一超额即拒绝：

1. User 级别（月度配额、RPM 限制）
2. Group 级别（用户所属组的共享配额）
3. Tenant 级别（租户总配额，多租户场景）
4. Global 级别（系统全局限制，防止过载）
```

### 3.2 配额配置（数据库）

```sql
-- 配额规则表
CREATE TABLE quota_rules (
    id          BIGSERIAL    PRIMARY KEY,
    dimension   VARCHAR(16)  NOT NULL, -- user | group | tenant | global
    target_id   VARCHAR(64),           -- user_id / group_id / tenant_id，global 时为 NULL
    period      VARCHAR(16)  NOT NULL, -- minute | hour | day | month | total
    limit_units BIGINT       NOT NULL, -- -1 = 无限
    enabled     BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (dimension, target_id, period)
);

-- 配额使用量表（按周期记录）
CREATE TABLE quota_usage (
    rule_id     BIGINT       NOT NULL REFERENCES quota_rules(id),
    period_key  VARCHAR(32)  NOT NULL, -- 周期标识，如 "2026-05"（月）、"2026-05-09"（日）
    used        BIGINT       NOT NULL DEFAULT 0,
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    PRIMARY KEY (rule_id, period_key)
);
```

### 3.3 RPM（每分钟请求数）限制

RPM 限制是特殊的配额规则（`period=minute`），推荐使用 Redis 实现以减少 DB 压力：

```go
// RPM 实现（Redis INCR + EXPIRE）：
//   key: quota:rpm:{dimension}:{targetID}:{minute_bucket}
//   INCR key
//   EXPIRE key 60
//   if value > limit: reject
```

<!-- @end-section -->

<!-- @section: config -->
---

## 4. 配置 Schema

```go
// QuotaBudgetManagerConfig 是 QuotaBudgetManager 的配置。
type QuotaBudgetManagerConfig struct {
    // Backend 配额存储后端。
    Backend BudgetBackend `validate:"required"`

    // RPMBackend RPM 限制的独立后端（推荐 Redis，可与 Backend 不同）。
    // 若为 nil，RPM 限制使用 Backend 实现（性能较低）。
    RPMBackend *BudgetBackend

    // CheckOrder 检查维度顺序（默认 user→group→tenant→global）。
    CheckOrder []BudgetDimension

    // CacheEnabled 是否缓存配额规则（减少 DB 查询）。
    CacheEnabled bool          `default:"true"`
    CacheTTL     time.Duration `default:"5m"`
}

type BudgetBackend struct {
    Type     string // "postgres" | "redis" | "memory"
    DSN      string // postgres 连接串
    RedisURL string // redis 连接串
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 5. 行为契约

| 契约 | 说明 |
|------|------|
| **Check 不扣减** | Check 是只读操作，不修改任何配额计数 |
| **Consume 幂等** | 相同 requestID 重复调用不重复扣减 |
| **Refund 幂等** | 相同 requestID 多次退还只执行一次 |
| **任一维度超额即拒绝** | 不做部分允许（全过或全拒）|
| **Global 维度不可绕过** | 即使 User/Group/Tenant 未超额，Global 超额也拒绝 |
| **无限配额（-1）不参与检查** | limit=-1 的规则直接跳过，不计算 Remaining |
| **Check 允许略旧** | 可使用缓存的配额规则，但 Consume 必须实时检查（防止超额） |

<!-- @end-section -->

<!-- @section: error-types -->
---

## 6. 错误类型

```go
// QuotaError 是 QuotaBudgetManager 方法返回的错误类型。
type QuotaError struct {
    Code      QuotaErrorCode
    Dimension BudgetDimension // 超额的维度
    Period    BudgetPeriod    // 超额的周期
    Limit     int64
    Used      int64
    Message   string
}

type QuotaErrorCode string

const (
    // ErrQuotaExceeded 配额超额，调用方返回 429。
    ErrQuotaExceeded QuotaErrorCode = "QUOTA_EXCEEDED"

    // ErrRPMLimitExceeded RPM 超额，调用方返回 429 并附 Retry-After。
    ErrRPMLimitExceeded QuotaErrorCode = "RPM_LIMIT_EXCEEDED"

    // ErrBudgetUnavailable 配额后端不可用（DB/Redis 故障）。
    // 调用方可选择降级（允许请求）或拒绝（保守策略）。
    ErrBudgetUnavailable QuotaErrorCode = "BUDGET_UNAVAILABLE"
)
```

<!-- @end-section -->

<!-- @section: testing -->
---

## 7. 测试支持

```go
// RunQuotaBudgetManagerContractTests 验证 QuotaBudgetManager 实现的行为契约。
func RunQuotaBudgetManagerContractTests(t *testing.T, mgr QuotaBudgetManager) {
    t.Run("Check/UnderLimit/Passes", ...)
    t.Run("Check/OverLimit/Rejects", ...)
    t.Run("Consume/Idempotent/SameRequestIDNoDuplicate", ...)
    t.Run("Refund/Idempotent/MultipleCallsOnlyRefundOnce", ...)
    t.Run("MultiDimension/AnyExceeds/Rejects", ...)
    t.Run("GlobalLimit/CannotBypass", ...)
    t.Run("Adjust/NegativeDelta/Refunds", ...)
    t.Run("ConcurrencySafety/ParallelConsume/NoOverflow", ...)
}

// NoopQuotaBudgetManager 是 C33 未注册时的默认 Noop 实现（始终允许）。
type NoopQuotaBudgetManager struct{}

func (n *NoopQuotaBudgetManager) Check(_ *RequestContext, _ int64) error       { return nil }
func (n *NoopQuotaBudgetManager) Consume(_ *RequestContext, _ string, _ int64) error { return nil }
func (n *NoopQuotaBudgetManager) Refund(_ *RequestContext, _ string) error     { return nil }
func (n *NoopQuotaBudgetManager) Adjust(_ *RequestContext, _ string, _ int64) error { return nil }
func (n *NoopQuotaBudgetManager) BudgetStatus(_ *RequestContext) BudgetStatusSnapshot {
    return BudgetStatusSnapshot{}
}
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 8. 实现检查清单

```
QuotaBudgetManager
  ☐ Check：只读，不扣减，允许使用缓存配额规则
  ☐ Consume：幂等（requestID 去重），实时检查（不用缓存）
  ☐ Refund：幂等（requestID 去重），多次退还只执行一次
  ☐ Adjust：支持正/负 delta
  ☐ 多维度顺序检查：任一超额即拒绝
  ☐ limit=-1 的规则跳过检查

RPM 限制
  ☐ 使用 Redis 实现（推荐），或 DB 实现（降级）
  ☐ 超额返回 ErrRPMLimitExceeded，含 Retry-After 时间

配额规则缓存
  ☐ CacheEnabled=true 时缓存规则（TTL 可配置）
  ☐ Consume 绕过缓存，直接查实时数据

并发安全
  ☐ 并发 Consume 不超额（DB 行锁或 Redis 原子操作）

Noop 实现
  ☐ C33 未注册时框架注入 NoopQuotaBudgetManager（始终允许）

测试
  ☐ 运行 RunQuotaBudgetManagerContractTests（全部通过）
  ☐ 并发 Consume 不超额（竞态测试）
  ☐ 多维度超额场景
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C33 在依赖图中的位置）
- quota-checker-spec.md（C13，在请求入口调用 Check）
- billing-session-spec.md（C31，在计费阶段调用 Consume / Refund / Adjust）
- circuit-breaker-spec.md（C34，可选依赖）

<!-- @end-section -->
