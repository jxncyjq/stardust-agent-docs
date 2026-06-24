---
id: "spec-component-model-router-014"
title: "ModelRouter 组件规范"
aliases: ["ModelRouter规范", "模型路由", "model-router-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "routing", "strategy", "maas", "interface"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C14"
layer: "L2"
depends_on:
  - "C03"   # ProviderRegistry — 提供候选提供商列表
  - "C04"   # ModelMapper — 将请求模型名映射到提供商模型名
optional_deps:
  - "C05"   # ProviderHealthMonitor — 缺失时不考虑健康分，均等路由
  - "C30"   # PricingEngine — 缺失时 CostAwareRouter 策略不可用
  - "C33"   # QuotaBudgetManager — 缺失时不考虑预算约束
conflicts_with: []
required_by: []
assembly_profiles:
  - standard
  - enterprise
  - embedded
---

<!-- @section: overview -->
# ModelRouter 组件规范

## 1. 组件定位

`ModelRouter` 在请求管道中负责**路由决策**：给定一个推理请求，从注册的提供商中选出最合适的 `(providerID, modelID)` 组合。

它采用**策略链**设计，每种路由策略是一个独立的 `RouterStrategy` 实现，按优先级依次执行，第一个返回非空结果的策略胜出。

```
请求 (ModelInput + 租户上下文)
        │
        ▼
  ModelRouter
  ┌─────────────────────────────────────┐
  │  策略链（按优先级从高到低）          │
  │  1. PinnedModelStrategy   (C24)     │ ← 管理员强制指定，最高优先级
  │  2. RoleBasedStrategy     (C20)     │
  │  3. TaskTypeStrategy      (C21)     │
  │  4. CostAwareStrategy     (C22)     │
  │  5. HealthAwareStrategy   (C23)     │ ← 兜底：基于健康分
  └─────────────────────────────────────┘
        │
        ▼ RouteResult{ProviderID, ModelID, Reason}
```

**读者**：配置路由策略的平台工程师、实现新策略的开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

### 2.1 ModelRouter 接口

```go
// ModelRouter 是请求管道中的路由决策组件。
// 并发安全：同一实例可在多个 goroutine 中并发调用。
type ModelRouter interface {
    // Route 从候选提供商中选出最合适的路由结果。
    // excludeProviders 是 FailoverManager 要求排除的提供商（已失败的）。
    // 返回 ErrNoRouteFound 时调用方应终止请求（无可用提供商）。
    Route(ctx *RequestContext, excludeProviders []string) (RouteResult, error)
}

// RouteResult 是路由决策结果。
type RouteResult struct {
    ProviderID  string   // 选中的提供商 ID
    ModelID     string   // 提供商侧的实际模型 ID（经 ModelMapper 转换）
    Strategy    string   // 命中的策略名（用于日志和遥测）
    Reason      string   // 人类可读的决策原因
    Alternatives []string // 备选提供商 ID（用于 FailoverManager 参考）
}

var ErrNoRouteFound = errors.New("no available provider matched routing criteria")
```

### 2.2 RouterStrategy 接口

```go
// RouterStrategy 是单个路由策略的接口。
// 所有内置策略（C20-C24）均实现此接口。
// 并发安全：策略实例在整个生命周期内被并发调用。
type RouterStrategy interface {
    // Name 返回策略唯一名称（kebab-case），用于日志和配置。
    Name() string

    // Priority 返回优先级数值，数值越小优先级越高。
    // 框架按 Priority 升序排列策略链。
    Priority() int

    // Match 判断此策略是否适用于当前请求上下文。
    // 返回 false 时策略链跳过此策略。
    Match(ctx *RequestContext) bool

    // Select 从候选列表中选择提供商。
    // candidates 是经过 ProviderRegistry 过滤和 excludeProviders 排除后的列表。
    // 返回 nil 表示此策略无法做出决策，继续下一个策略。
    Select(ctx *RequestContext, candidates []ProviderCandidate) *RouteResult
}

// ProviderCandidate 是路由候选项（含路由决策所需的元数据）。
type ProviderCandidate struct {
    ProviderID    string
    ModelID       string            // 提供商侧模型 ID
    HealthScore   float64           // 0.0~1.0，来自 ProviderHealthMonitor
    InputPrice    float64           // USD per 1M tokens
    OutputPrice   float64
    ReasoningLevel ReasoningLevel   // LOW | MEDIUM | HIGH | ULTRA
    Modalities    []Modality
}
```

<!-- @end-section -->

<!-- @section: strategies -->
---

## 3. 内置策略详解

### 3.1 PinnedModelStrategy（C24，Priority=0）

管理员通过管理后台或配置强制指定某 Agent / 租户使用特定模型，覆盖所有其他策略。

```go
// PinnedModelStrategy 从 RequestContext.TenantContext 中读取强制绑定配置。
// 配置来源：数据库 pinned_routes 表（管理后台写入）。
//
// 配置结构：
//   key:   agent_id | tenant_id | "*"（全局）
//   value: { provider_id: "anthropic", model_id: "claude-opus-4-7" }
//
// Match 条件：RequestContext 中存在匹配的强制绑定配置
// Select 行为：直接返回配置中的 provider 和 model，忽略候选列表
```

### 3.2 RoleBasedStrategy（C20，Priority=10）

按 Agent 角色匹配预定义的模型偏好。

```go
// 配置（YAML）：
//   role_routing:
//     ceo:              { prefer: "claude-opus-4-7",  fallback: "gpt-4o" }
//     developer:        { prefer: "claude-sonnet-4-6", fallback: "deepseek-coder" }
//     customer_service: { prefer: "deepseek-chat",    fallback: "qwen-turbo" }
//
// Match 条件：RequestContext.AgentRole 非空
// Select 行为：
//   1. 在候选列表中找 prefer 模型 → 找到则选择
//   2. prefer 不在候选中 → 找 fallback 模型
//   3. 均不在候选中 → 返回 nil（下一策略处理）
```

### 3.3 TaskTypeStrategy（C21，Priority=20）

按任务类型匹配能力要求，比角色更精确。

```go
// 配置（YAML）：
//   task_routing:
//     strategic_planning:
//       min_reasoning_level: HIGH
//       prefer_cost: false
//     code_generation:
//       require_modality: CODE
//       prefer_cost: true
//     data_extraction:
//       min_reasoning_level: LOW
//       prefer_cost: true
//
// Match 条件：RequestContext.TaskType 非空且在配置中有定义
// Select 行为：
//   1. 过滤候选：ReasoningLevel >= min_reasoning_level，包含 require_modality
//   2. prefer_cost=true → 按 InputPrice+OutputPrice 升序选最便宜
//   3. prefer_cost=false → 按 ReasoningLevel 降序选最强
//   4. 过滤后候选为空 → 返回 nil
```

### 3.4 CostAwareStrategy（C22，Priority=30）

在满足质量门槛的前提下，按成本优先选择。需要 `PricingEngine（C30）`。

```go
// Match 条件：始终 true（兜底策略之一）
// Select 行为：
//   1. 过滤掉 HealthScore < 0.5 的提供商（不稳定的不选）
//   2. 按 InputPrice × 0.7 + OutputPrice × 0.3 加权成本升序排列
//   3. 选成本最低的提供商
//   4. 可选：预算约束检查（QuotaBudgetManager，C33）
//
// 当 C30 未注册时：此策略的 Match() 始终返回 false（自动降级）
```

### 3.5 HealthAwareStrategy（C23，Priority=40）

按提供商健康分路由，作为最终兜底。需要 `ProviderHealthMonitor（C05）`。

```go
// Match 条件：始终 true
// Select 行为：
//   1. 过滤掉 HealthScore == 0 的提供商（已熔断）
//   2. 按 HealthScore 降序选最健康的提供商
//   3. HealthScore 相同时随机打散（避免所有请求打到同一提供商）
//
// 当 C05 未注册时：所有候选 HealthScore 均为 1.0，退化为随机路由
```

<!-- @end-section -->

<!-- @section: strategy-chain -->
---

## 4. 策略链执行规则

### 4.1 执行流程

```
1. ProviderRegistry.ListHealthy() → 全量候选列表
2. ModelMapper.Map(requestedModel) → 过滤支持该模型的候选
3. 排除 excludeProviders → 剩余候选
4. 按 Priority 升序遍历策略链：
     for each strategy:
       if strategy.Match(ctx):
         result = strategy.Select(ctx, candidates)
         if result != nil: return result  ← 第一个非空结果胜出
5. 所有策略均返回 nil → ErrNoRouteFound
```

### 4.2 策略链配置

框架支持在配置文件中自定义策略链：

```yaml
model_router:
  strategy_chain:
    - name: pinned          # C24，不可禁用
      priority: 0
    - name: role_based      # C20
      priority: 10
      enabled: true
    - name: task_type       # C21
      priority: 20
      enabled: true
    - name: cost_aware      # C22
      priority: 30
      enabled: true
    - name: health_aware    # C23，不可禁用（兜底）
      priority: 40
```

### 4.3 自定义策略注册

```go
// 注册自定义策略（在 ComponentRegistry.Build 之前调用）
registry.RegisterRouterStrategy(&myCustomStrategy{})

// 自定义策略只需实现 RouterStrategy 接口，框架自动插入策略链
```

<!-- @end-section -->

<!-- @section: config -->
---

## 5. 配置 Schema

```go
// ModelRouterConfig 是 ModelRouter 的配置。
type ModelRouterConfig struct {
    // StrategyChain 策略链配置（按 Priority 升序排列）。
    // 若为空，使用默认配置（所有内置策略按默认优先级）。
    StrategyChain []StrategyConfig

    // FallbackToAnyProvider 当所有策略均无法选择时，是否降级到随机选择。
    // false（默认）= 返回 ErrNoRouteFound。
    // true = 从候选中随机选一个（最大可用性，降低路由质量）。
    FallbackToAnyProvider bool `default:"false"`

    // CandidateHealthThreshold 过滤候选时的最低健康分。
    // 低于此分的提供商不进入候选列表（需要 C05 已注册）。
    CandidateHealthThreshold float64 `default:"0.1" validate:"min=0,max=1"`
}

// StrategyConfig 单个策略的配置。
type StrategyConfig struct {
    Name     string         // 策略名，对应 RouterStrategy.Name()
    Priority int            // 覆盖默认优先级
    Enabled  bool           `default:"true"`
    Options  map[string]any // 传递给策略的额外配置
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 6. 行为契约

| 契约 | 说明 |
|------|------|
| **幂等路由（软约束）** | 相同请求上下文 + 相同候选列表，策略链应倾向于返回相同结果（`HealthAwareStrategy` 的随机打散除外） |
| **不修改候选列表** | `Select()` 接收的 `candidates` 是只读切片，不得修改 |
| **Strategy 并发安全** | 同一 Strategy 实例被多 goroutine 并发调用，不得有共享可变状态 |
| **PinnedModel 最高优先** | `PinnedModelStrategy` 的优先级必须为 0，框架强制校验 |
| **HealthAware 不能禁用** | 保证至少有一个兜底策略，框架强制校验 |
| **Route 不做 IO** | `Route()` 的候选列表来自内存缓存，不得在路由路径上查询 DB |
| **失败原因可追溯** | `RouteResult.Reason` 必须说明命中的策略和决策依据 |

<!-- @end-section -->

<!-- @section: error-types -->
---

## 7. 错误类型

```go
// RouterError 是 ModelRouter 返回的错误类型。
type RouterError struct {
    Code    RouterErrorCode
    Message string
}

type RouterErrorCode string

const (
    // ErrNoRouteFound 无可用提供商（所有策略均返回 nil）。
    // 调用方应返回 503。
    ErrNoRouteFound RouterErrorCode = "NO_ROUTE_FOUND"

    // ErrModelNotSupported 请求的模型不被任何已注册提供商支持。
    // 调用方应返回 400。
    ErrModelNotSupported RouterErrorCode = "MODEL_NOT_SUPPORTED"

    // ErrBudgetExhausted 预算约束导致无法路由（QuotaBudgetManager 返回禁止）。
    // 调用方应返回 402。
    ErrBudgetExhausted RouterErrorCode = "BUDGET_EXHAUSTED"
)
```

<!-- @end-section -->

<!-- @section: testing -->
---

## 8. 测试支持

### 8.1 契约测试

```go
// RunModelRouterContractTests 验证路由策略链和接口契约。
func RunModelRouterContractTests(t *testing.T, router ModelRouter) {
    t.Run("PinnedStrategy/AlwaysWinsOverOthers", ...)
    t.Run("ExcludeProviders/NeverReturnsExcluded", ...)
    t.Run("AllStrategiesReturnNil/ReturnsErrNoRouteFound", ...)
    t.Run("EmptyCandidates/ReturnsErrNoRouteFound", ...)
    t.Run("ConcurrencySafety/ParallelRoute", ...)
    t.Run("RouteResult/ReasonNonEmpty", ...)
}
```

### 8.2 策略单元测试模板

```go
// 每个 RouterStrategy 实现需覆盖以下用例：
//   TestStrategy_Match_ReturnsTrue_WhenContextHasRole
//   TestStrategy_Match_ReturnsFalse_WhenContextLacksRole
//   TestStrategy_Select_ReturnsNil_WhenNoCandidateMatchesPrefer
//   TestStrategy_Select_ReturnsPrefer_WhenAvailable
//   TestStrategy_Select_ReturnsFallback_WhenPreferUnavailable
//   TestStrategy_ConcurrencySafe
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 9. 实现检查清单

```
ModelRouter
  ☐ 按 Priority 升序执行策略链
  ☐ 第一个非 nil 结果立即返回
  ☐ excludeProviders 在进入策略链前过滤
  ☐ CandidateHealthThreshold 过滤（若 C05 已注册）
  ☐ FallbackToAnyProvider 降级逻辑（若配置为 true）
  ☐ Route 不做 IO

PinnedModelStrategy（C24）
  ☐ Priority = 0，框架启动时验证
  ☐ 从 RequestContext.TenantContext 读取绑定配置

RoleBasedStrategy（C20）
  ☐ prefer → fallback 两级查找
  ☐ 两级均不在候选中时返回 nil

TaskTypeStrategy（C21）
  ☐ 能力过滤：ReasoningLevel + Modality
  ☐ prefer_cost=true → 按价格升序
  ☐ prefer_cost=false → 按能力降序

CostAwareStrategy（C22）
  ☐ C30 未注册时 Match() 返回 false
  ☐ 健康分 < 0.5 的过滤

HealthAwareStrategy（C23）
  ☐ 不可被禁用，框架启动时验证
  ☐ HealthScore 相同时随机打散

测试
  ☐ 运行 RunModelRouterContractTests（全部通过）
  ☐ 每个内置策略覆盖单元测试模板
  ☐ 并发安全验证
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C14 在依赖图中的位置）
- provider-registry-spec.md（C03，必须依赖）
- model-mapper-spec.md（C04，必须依赖）
- provider-health-monitor-spec.md（C05，可选依赖）
- pricing-engine-spec.md（C30，CostAwareStrategy 依赖）
- quota-budget-manager-spec.md（C33，可选依赖）
- [[model-provider-spec|ModelProvider 组件规范]]（C01）

<!-- @end-section -->
