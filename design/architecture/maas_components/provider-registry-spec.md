---
id: "spec-component-provider-registry-003"
title: "ProviderRegistry 组件规范"
aliases: ["ProviderRegistry规范", "提供商注册表", "provider-registry-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "registry", "provider", "maas", "interface"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C03"
layer: "L1"
depends_on:
  - "C01"   # ModelProvider — 注册表聚合 ModelProvider 实例
optional_deps:
  - "C05"   # ProviderHealthMonitor — 缺失时 ListHealthy 返回全量列表
conflicts_with: []
required_by:
  - "C14"   # ModelRouter 从注册表获取候选提供商列表
assembly_profiles:
  - minimal
  - standard
  - enterprise
  - embedded
---

<!-- @section: overview -->
# ProviderRegistry 组件规范

## 1. 组件定位

`ProviderRegistry` 是提供商层的**中央注册表**，聚合所有已注册的 `ModelProvider` 实例，并向路由层提供经过健康过滤的候选列表。

它是 `ModelRouter` 的必须依赖，负责将"哪些提供商可用"的问题与路由决策逻辑解耦。

```
框架启动
  │ Register(provider)
  ▼
ProviderRegistry
  ┌────────────────────────────────────────────┐
  │  providers: map[string]ModelProvider        │
  │  healthMonitor: ProviderHealthMonitor (可选) │
  │                                            │
  │  Get(providerID) → ModelProvider            │
  │  ListAll()       → []ModelProvider          │
  │  ListHealthy()   → []ProviderCandidate      │ ← ModelRouter 调用
  │  ListByModel(modelID) → []ProviderCandidate │
  └────────────────────────────────────────────┘
        │
        ▼ ModelRouter 使用 ProviderCandidate 列表路由
```

**读者**：配置提供商的平台工程师、使用注册表的路由开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

### 2.1 ProviderRegistry 接口

```go
// ProviderRegistry 聚合并管理所有已注册的 ModelProvider 实例。
// 并发安全：注册操作仅在启动阶段执行，运行期只读。
type ProviderRegistry interface {
    // Register 注册一个 ModelProvider 实现。
    // 仅在框架启动阶段（Build 之前）调用，运行期不支持动态注册。
    // 同一 providerID 重复注册时 panic（启动时配置错误应快速失败）。
    Register(provider ModelProvider)

    // Get 通过 providerID 获取对应的 ModelProvider 实例。
    // providerID 不存在时返回 nil, ErrProviderNotFound。
    Get(providerID string) (ModelProvider, error)

    // ListAll 返回所有已注册的提供商（不过滤健康状态）。
    // 用于管理接口和监控，不用于路由决策。
    ListAll() []ModelProvider

    // ListHealthy 返回健康状态满足阈值的提供商候选列表。
    // threshold 是最低健康分（由 ModelRouter 的 CandidateHealthThreshold 配置传入）。
    // 若 ProviderHealthMonitor 未注册，所有提供商的 HealthScore 视为 1.0。
    ListHealthy(threshold float64) []ProviderCandidate

    // ListByModel 返回支持指定模型（框架侧名称）的健康提供商候选列表。
    // 内部通过 ModelMapper 将框架模型名转换为提供商侧模型名。
    // 结合 ListHealthy 的过滤逻辑，threshold 同上。
    ListByModel(modelID string, threshold float64) []ProviderCandidate
}
```

### 2.2 ProviderCandidate

```go
// ProviderCandidate 是路由候选项，聚合了路由决策所需的所有元数据。
// 由 ProviderRegistry 在 ListHealthy / ListByModel 时组装，避免路由层直接访问多个组件。
type ProviderCandidate struct {
    ProviderID     string
    ModelID        string          // 提供商侧实际模型 ID（经 ModelMapper 转换）
    Provider       ModelProvider   // 原始实现引用（ProviderExecutor 用于实际调用）
    HealthScore    float64         // 0.0~1.0，来自 ProviderHealthMonitor（缺失时为 1.0）
    InputPrice     float64         // USD per 1M input tokens（来自 PricingEngine.ModelPrice）
    OutputPrice    float64         // USD per 1M output tokens
    ReasoningLevel ReasoningLevel  // LOW | MEDIUM | HIGH | ULTRA
    Modalities     []Modality      // TEXT | CODE | IMAGE | AUDIO | VIDEO
    RPMLimit       int             // 来自 ProviderConfig，0 表示不限
    ConcurrencyMax int
}
```

### 2.3 相关枚举

```go
// ReasoningLevel 模型推理能力等级，用于 TaskTypeStrategy 的能力过滤。
type ReasoningLevel int

const (
    ReasoningLevelLow    ReasoningLevel = iota // 基础对话模型
    ReasoningLevelMedium                       // 标准推理模型（如 claude-sonnet）
    ReasoningLevelHigh                         // 高级推理模型（如 claude-opus）
    ReasoningLevelUltra                        // 扩展思考模型（如带 thinking 的 claude-opus）
)

// Modality 模型支持的模态。
type Modality string

const (
    ModalityText  Modality = "TEXT"
    ModalityCode  Modality = "CODE"
    ModalityImage Modality = "IMAGE"
    ModalityAudio Modality = "AUDIO"
    ModalityVideo Modality = "VIDEO"
)
```

<!-- @end-section -->

<!-- @section: model-capabilities -->
---

## 3. 模型能力注册

`ProviderRegistry` 从 `model_capabilities` 表（数据库）或配置文件加载每个提供商 + 模型的能力元数据：

```sql
-- 模型能力表（框架启动时加载到内存）
CREATE TABLE model_capabilities (
    id              BIGSERIAL    PRIMARY KEY,
    provider_id     VARCHAR(64)  NOT NULL,  -- 对应 ModelProvider.ID()
    provider_model  VARCHAR(128) NOT NULL,  -- 提供商侧模型 ID
    framework_model VARCHAR(128) NOT NULL,  -- 框架统一模型名（ModelMapper 来源）
    reasoning_level VARCHAR(16)  NOT NULL DEFAULT 'MEDIUM',
    modalities      TEXT[]       NOT NULL DEFAULT '{TEXT}',
    input_price_usd NUMERIC(10,6),          -- USD per 1M tokens，可为 NULL（从 PricingEngine 获取）
    output_price_usd NUMERIC(10,6),
    rpm_limit       INT          NOT NULL DEFAULT 0,
    concurrency_max INT          NOT NULL DEFAULT 10,
    enabled         BOOLEAN      NOT NULL DEFAULT TRUE,
    UNIQUE (provider_id, provider_model)
);
```

**YAML 配置方式**（替代数据库）：

```yaml
model_capabilities:
  - provider_id: anthropic-primary
    provider_model: claude-opus-4-7
    framework_model: claude-opus-4-7
    reasoning_level: ULTRA
    modalities: [TEXT, CODE]
    rpm_limit: 100
    concurrency_max: 20

  - provider_id: openai-primary
    provider_model: gpt-4o
    framework_model: gpt-4o
    reasoning_level: HIGH
    modalities: [TEXT, CODE, IMAGE]
    rpm_limit: 500
    concurrency_max: 50
```

<!-- @end-section -->

<!-- @section: model-alias -->
---

## 4. 模型别名（ModelMapper 集成）

`ListByModel` 接受框架侧模型名（如 `claude-opus-4-7`），内部使用 `ModelMapper` 解析别名：

```
请求: model="claude-3-opus"  （用户传入的别名）
        │
        ▼ ModelMapper.Resolve("claude-3-opus")
        │
        ▼ "claude-opus-4-7"  （框架统一名）
        │
        ▼ ProviderRegistry.ListByModel("claude-opus-4-7", threshold)
        │
        ▼ → anthropic-primary(claude-opus-4-7), openai-azure(claude-opus-4-7)
```

`ModelMapper`（C04）不是 `ProviderRegistry` 的必须依赖，而是由框架在构建 `ModelRouter` 时注入：

```go
// ModelRouter 在调用 ListByModel 前已完成别名解析，
// ProviderRegistry 接收的 modelID 始终是框架统一名。
```

<!-- @end-section -->

<!-- @section: config -->
---

## 5. 配置 Schema

```go
// ProviderRegistryConfig 是 ProviderRegistry 的配置。
type ProviderRegistryConfig struct {
    // CapabilitiesSource 模型能力数据来源。
    CapabilitiesSource CapabilitiesSource `validate:"required"`

    // DefaultHealthScore 当 ProviderHealthMonitor 未注册时使用的默认健康分。
    DefaultHealthScore float64 `default:"1.0" validate:"min=0,max=1"`

    // PriceFallback 当 PricingEngine 未提供价格时使用的默认值。
    // 0 = 不在候选中填入价格（CostAwareStrategy 将忽略此候选）。
    PriceFallback float64 `default:"0"`
}

type CapabilitiesSource struct {
    Type     string // "yaml" | "database"
    FilePath string // type=yaml 时
    Table    string // type=database 时（默认 "model_capabilities"）
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 6. 行为契约

| 契约 | 说明 |
|------|------|
| **Register 仅限启动阶段** | 运行期不支持动态注册；热更新通过 Reload 实现（不在此版本） |
| **重复 ID 快速失败** | 同一 providerID 注册两次时 panic，暴露配置错误 |
| **ListHealthy 不做 IO** | 健康分来自内存缓存（HealthMonitor 定期刷新），不实时查询 |
| **ListByModel 不做 IO** | 能力数据在启动时加载到内存，运行期只读 |
| **HealthScore 缺省为 1.0** | C05 未注册时，所有候选 HealthScore 均为 1.0 |
| **价格缺省为 0** | PricingEngine 未提供价格时，InputPrice/OutputPrice 为 0，CostAwareStrategy 将跳过此候选 |
| **候选列表不可修改** | ListHealthy / ListByModel 返回的切片是独立副本，调用方修改不影响注册表 |

<!-- @end-section -->

<!-- @section: error-types -->
---

## 7. 错误类型

```go
var (
    // ErrProviderNotFound 指定 providerID 未注册。
    ErrProviderNotFound = errors.New("provider not found")

    // ErrModelNotSupported 没有任何已注册提供商支持该模型。
    // 由 ListByModel 在过滤后候选为空时返回（可选，也可返回空切片）。
    ErrModelNotSupported = errors.New("model not supported by any registered provider")
)
```

<!-- @end-section -->

<!-- @section: testing -->
---

## 8. 测试支持

### 8.1 契约测试

```go
// RunProviderRegistryContractTests 验证 ProviderRegistry 实现的行为契约。
func RunProviderRegistryContractTests(t *testing.T, registry ProviderRegistry) {
    t.Run("Register/DuplicateIDPanics", ...)
    t.Run("Get/NotFoundReturnsError", ...)
    t.Run("ListHealthy/FiltersBelow Threshold", ...)
    t.Run("ListHealthy/NoHealthMonitor/AllScore1", ...)
    t.Run("ListByModel/FiltersUnsupportedModels", ...)
    t.Run("ListHealthy/ReturnsCopy/MutationSafe", ...)
    t.Run("ConcurrencySafety/ParallelListHealthy", ...)
}
```

### 8.2 测试用注册表构建器

```go
// NewTestRegistry 构建一个用于测试的 ProviderRegistry。
// 支持直接传入 ProviderCandidate 覆盖，跳过数据库和 HealthMonitor。
func NewTestRegistry(candidates ...ProviderCandidate) ProviderRegistry {
    r := &inMemoryRegistry{}
    for _, c := range candidates {
        r.candidates[c.ProviderID] = c
    }
    return r
}

// 使用示例：
func TestModelRouter_WithMultipleProviders(t *testing.T) {
    registry := NewTestRegistry(
        ProviderCandidate{ProviderID: "anthropic", ModelID: "claude-opus-4-7", HealthScore: 0.9},
        ProviderCandidate{ProviderID: "openai", ModelID: "gpt-4o", HealthScore: 0.5},
    )
    router := NewModelRouter(registry, nil)
    // ...
}
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 9. 实现检查清单

```
ProviderRegistry
  ☐ Register：重复 providerID 时 panic（启动阶段快速失败）
  ☐ Get：providerID 不存在时返回 ErrProviderNotFound
  ☐ ListHealthy：按 threshold 过滤 HealthScore，C05 缺失时默认 1.0
  ☐ ListByModel：按 frameworkModelID 过滤能力表，结合健康过滤
  ☐ 候选列表是独立副本（调用方修改安全）
  ☐ 所有读方法并发安全

模型能力加载
  ☐ 支持 YAML 和数据库两种来源
  ☐ 启动时加载到内存，运行期不做 IO
  ☐ enabled=false 的能力行不进入候选

ProviderCandidate 组装
  ☐ HealthScore 从 C05 获取（缺失时 1.0）
  ☐ InputPrice / OutputPrice 从 C30 的 ModelPrice() 获取（缺失时 0）
  ☐ ReasoningLevel / Modalities 来自 model_capabilities 表

测试
  ☐ 运行 RunProviderRegistryContractTests（全部通过）
  ☐ 健康分过滤边界（threshold 临界值）
  ☐ 并发读安全验证
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C03 在依赖图中的位置）
- model-provider-spec.md（C01，必须依赖）
- provider-health-monitor-spec.md（C05，可选依赖）
- model-mapper-spec.md（C04，框架在路由层集成，非直接依赖）
- model-router-spec.md（C14，主要消费者）

<!-- @end-section -->
