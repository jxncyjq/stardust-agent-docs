---
id: "spec-component-registry-000"
title: "组件注册表与依赖关系图"
aliases: ["组件注册表", "component-registry", "依赖图"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "registry", "dependency", "assembly", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-llm-infra-abstraction-001"
---

<!-- @section: overview -->
# 组件注册表与依赖关系图

## 文档目的

本文档是所有组件规范的**目录和依赖主表**。它定义：

1. 每个组件的唯一 ID、层次归属、接口路径
2. 组件间的依赖关系（必须依赖 / 可选依赖 / 互斥关系）
3. 最小可用集与按场景的装配方案
4. 各组件规范文档的索引

**读者**：架构师、平台工程师、需要理解装配逻辑的实现者。

<!-- @end-section -->

<!-- @section: dependency-fields -->
---

## 一、依赖字段规范

每份组件规范文档的 Front Matter 中必须包含以下字段（本注册表是权威来源）：

```yaml
# 必须依赖：启动时若未注册这些组件，框架拒绝启动
depends_on:
  - component-id-a
  - component-id-b

# 可选依赖：未注册时框架使用内置 Noop 实现（降级运行）
optional_deps:
  - component-id-c   # 缺失时的默认行为描述

# 被以下组件依赖（反向索引，由注册表维护，规范文档无需手动填写）
required_by:
  - component-id-x

# 与以下组件互斥（同时注册会导致启动失败）
conflicts_with:
  - component-id-y
```

**框架在启动时执行依赖校验**：
1. 拓扑排序所有已注册组件，检测循环依赖
2. 验证所有 `depends_on` 的组件均已注册
3. 报告缺失的必须依赖，提示可安装的最小集

<!-- @end-section -->

<!-- @section: component-table -->
---

## 二、组件主表

### 层次说明

```
L0  基础类型层    — 纯数据结构和接口定义，无运行时依赖
L1  提供商层      — 对接外部 AI 服务
L2  管道层        — 请求处理流水线的各节点
L3  路由策略层    — 智能路由决策
L4  计费层        — 成本核算与配额管控
L5  可靠性层      — 故障转移与重试
L6  流式处理层    — SSE / 流式代理
L7  可观测层      — 追踪、指标、审计
L8  平台推理端口  — 面向 Agent/Know/Aegis 的稳定推理门面
```

### 完整组件表

| ID | 组件名 | 层次 | 接口路径 | 规范文档 | 状态 |
|----|--------|------|----------|----------|------|
| `C00` | **ComponentRegistry** | L0 | `component.Registry` | [component-registry-impl-spec.md](./component-registry-impl-spec.md) | ✅ |
| `C01` | **ModelProvider** | L1 | `provider.ModelProvider` | [model-provider-spec.md](./model-provider-spec.md) | ✅ |
| `C02` | **AsyncTaskProvider** | L1 | `provider.AsyncTaskProvider` | [async-task-provider-spec.md](./async-task-provider-spec.md) | ✅ |
| `C03` | **ProviderRegistry** | L1 | `provider.Registry` | [provider-registry-spec.md](./provider-registry-spec.md) | ✅ |
| `C04` | **ModelMapper** | L1 | `provider.ModelMapper` | [model-mapper-spec.md](./model-mapper-spec.md) | ✅ |
| `C05` | **ProviderHealthMonitor** | L1 | `provider.HealthMonitor` | [provider-health-monitor-spec.md](./provider-health-monitor-spec.md) | ✅ |
| `C06` | **ConcurrencyLimiter** | L1 | `provider.ConcurrencyLimiter` | [concurrency-limiter-spec.md](./concurrency-limiter-spec.md) | ✅ |
| `C10` | **AuthGate** | L2 | `pipeline.AuthGate` | [auth-gate-spec.md](./auth-gate-spec.md) | ✅ |
| `C11` | **TenantContextLoader** | L2 | `pipeline.TenantContextLoader` | [tenant-context-loader-spec.md](./tenant-context-loader-spec.md) | ✅ |
| `C12` | **RateLimiter** | L2 | `pipeline.RateLimiter` | [rate-limiter-spec.md](./rate-limiter-spec.md) | ✅ |
| `C13` | **QuotaChecker** | L2 | `pipeline.QuotaChecker` | [quota-checker-spec.md](./quota-checker-spec.md) | ✅ |
| `C14` | **ModelRouter** | L2 | `pipeline.ModelRouter` | [model-router-spec.md](./model-router-spec.md) | ✅ |
| `C15` | **RequestTransformer** | L2 | `pipeline.RequestTransformer` | — 内置于 ModelProvider | — |
| `C16` | **SemanticCache** | L2 | `pipeline.SemanticCache` | [semantic-cache-spec.md](./semantic-cache-spec.md) | ✅ |
| `C17` | **ContentFilter** | L2 | `pipeline.ContentFilter` | [content-filter-spec.md](./content-filter-spec.md) | ✅ |
| `C18` | **ResponseNormalizer** | L2 | `pipeline.ResponseNormalizer` | — 内置于 ModelProvider | — |
| `C19` | **TelemetryEmitter** | L2 | `pipeline.TelemetryEmitter` | [telemetry-emitter-spec.md](./telemetry-emitter-spec.md) | ✅ |
| `C20` | **RoleBasedRouter** | L3 | `router.RoleBasedStrategy` | — 内置于 ModelRouter | — |
| `C21` | **TaskTypeRouter** | L3 | `router.TaskTypeStrategy` | — 内置于 ModelRouter | — |
| `C22` | **CostAwareRouter** | L3 | `router.CostAwareStrategy` | — 内置于 ModelRouter | — |
| `C23` | **HealthAwareRouter** | L3 | `router.HealthAwareStrategy` | — 内置于 ModelRouter | — |
| `C24` | **PinnedModelRouter** | L3 | `router.PinnedModelStrategy` | — 内置于 ModelRouter | — |
| `C30` | **PricingEngine** | L4 | `billing.PricingEngine` | [pricing-engine-spec.md](./pricing-engine-spec.md) | ✅ |
| `C31` | **BillingSession** | L4 | `billing.Session` | [billing-session-spec.md](./billing-session-spec.md) | ✅ |
| `C32` | **FundingSource** | L4 | `billing.FundingSource` | [funding-source-spec.md](./funding-source-spec.md) | ✅ |
| `C33` | **QuotaBudgetManager** | L4 | `billing.QuotaBudgetManager` | [quota-budget-manager-spec.md](./quota-budget-manager-spec.md) | ✅ |
| `C34` | **CircuitBreaker** | L4 | `billing.CircuitBreaker` | [circuit-breaker-spec.md](./circuit-breaker-spec.md) | ✅ |
| `C40` | **FailoverManager** | L5 | `reliability.FailoverManager` | [failover-manager-spec.md](./failover-manager-spec.md) | ✅ |
| `C41` | **RetryScheduler** | L5 | `reliability.RetryScheduler` | [retry-scheduler-spec.md](./retry-scheduler-spec.md) | ✅ |
| `C42` | **ProviderErrorClassifier** | L5 | `reliability.ErrorClassifier` | — 内置于 ModelProvider | — |
| `C50` | **StreamProxy** | L6 | `stream.Proxy` | [stream-proxy-spec.md](./stream-proxy-spec.md) | ✅ |
| `C51` | **StreamAggregator** | L6 | `stream.Aggregator` | — 内置于 StreamProxy | — |
| `C52` | **StreamUsageExtractor** | L6 | `stream.UsageExtractor` | — 内置于 ModelProvider | — |
| `C60` | **TraceCollector** | L7 | `observe.TraceCollector` | [trace-collector-spec.md](./trace-collector-spec.md) | ✅ |
| `C61` | **MetricsRecorder** | L7 | `observe.MetricsRecorder` | [metrics-recorder-spec.md](./metrics-recorder-spec.md) | ✅ |
| `C62` | **AuditLogger** | L7 | `observe.AuditLogger` | [audit-logger-spec.md](./audit-logger-spec.md) | ✅ |
| `C63` | **CostAttributor** | L7 | `observe.CostAttributor` | [cost-attributor-spec.md](./cost-attributor-spec.md) | ✅ |
| `C70` | **MaasInferenceClient** | L8 | `inference.Client` | [maas-inference-client-spec.md](./maas-inference-client-spec.md) | ✅ |

<!-- @end-section -->

<!-- @section: dependency-graph -->
---

## 三、依赖关系图

```
L0  ComponentRegistry (C00)
        │ 被所有组件依赖（隐式）
        │
L1  ┌───┴─────────────────────────────────────────┐
    │                                             │
    C03 ProviderRegistry ←── C01 ModelProvider    C04 ModelMapper
    │        │                    │               │
    │        │                    └── C05 ProviderHealthMonitor
    │        │                    └── C06 ConcurrencyLimiter
    │        │
L2  ├── C10 AuthGate
    ├── C11 TenantContextLoader
    ├── C12 RateLimiter ──────────── (可选) C06 ConcurrencyLimiter
    ├── C13 QuotaChecker ─────────── (必须) C33 QuotaBudgetManager
    ├── C14 ModelRouter ──────────── (必须) C03 ProviderRegistry
    │        │                              C04 ModelMapper
    │        │                       (可选) C33 QuotaBudgetManager
    │        │                              C05 ProviderHealthMonitor
    ├── C16 SemanticCache
    ├── C17 ContentFilter
    └── C19 TelemetryEmitter ──────── (可选) C60 TraceCollector
                                              C61 MetricsRecorder

L3  路由策略（均内置于 C14 ModelRouter，通过策略链组合）
    C20 RoleBasedRouter
    C21 TaskTypeRouter
    C22 CostAwareRouter ─────────── (必须) C30 PricingEngine
    C23 HealthAwareRouter ────────── (必须) C05 ProviderHealthMonitor
    C24 PinnedModelRouter

L4  C30 PricingEngine
    C31 BillingSession ──────────── (必须) C30 PricingEngine
    │                                       C32 FundingSource
    │                               (可选) C33 QuotaBudgetManager
    C32 FundingSource               (无依赖，独立组件)
    C33 QuotaBudgetManager ─────── (可选) C34 CircuitBreaker
    C34 CircuitBreaker

L5  C40 FailoverManager ─────────── (必须) C41 RetryScheduler
    │                               (可选) C05 ProviderHealthMonitor
    C41 RetryScheduler              (无依赖)

L6  C50 StreamProxy ─────────────── (必须) C01 ModelProvider（通过框架传入）

L7  C60 TraceCollector             (无依赖，输出到外部系统)
    C61 MetricsRecorder            (无依赖)
    C62 AuditLogger                (无依赖)
    C63 CostAttributor ─────────── (必须) C31 BillingSession

L8  C70 MaasInferenceClient ────── (必须) C01 ModelProvider
                                             C14 ModelRouter
                                    (可选) C16 SemanticCache
                                             C17 ContentFilter
                                             C19 TelemetryEmitter
                                             C40 FailoverManager
                                             C50 StreamProxy
                                             C62 AuditLogger
```

### 依赖矩阵（精简版）

> `●` = 必须依赖，`○` = 可选依赖，空 = 无依赖

| 组件 \ 依赖 | C01 | C03 | C04 | C05 | C06 | C14 | C16 | C17 | C19 | C30 | C31 | C32 | C33 | C34 | C40 | C41 | C50 | C60 | C61 | C62 |
|-------------|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|
| C03 ProviderRegistry | ● | | | | | | | | | | | | | | | | | | | |
| C04 ModelMapper | | | | | | | | | | | | | | | | | | | | |
| C05 HealthMonitor | ● | | | | | | | | | | | | | | | | | | | |
| C06 ConcurrencyLimiter | ● | | | | | | | | | | | | | | | | | | | |
| C12 RateLimiter | | | | | ○ | | | | | | | | | | | | | | | |
| C13 QuotaChecker | | | | | | | | | | | | | ● | | | | | | | |
| C14 ModelRouter | | ● | ● | ○ | | | | | | ○ | | | ○ | | | | | | | |
| C19 TelemetryEmitter | | | | | | | | | | | | | | | | | | ○ | ○ | |
| C22 CostAwareRouter | | | | | | | | | | ● | | | | | | | | | | |
| C23 HealthAwareRouter | | | | ● | | | | | | | | | | | | | | | | |
| C31 BillingSession | | | | | | | | | | ● | | ● | ○ | | | | | | | |
| C33 QuotaBudgetManager | | | | | | | | | | | | | | ○ | | | | | | |
| C40 FailoverManager | | | | ○ | | | | | | | | | | | | ● | | | | |
| C50 StreamProxy | ● | | | | | | | | | | | | | | | | | | | |
| C63 CostAttributor | | | | | | | | | | | ● | | | | | | | | | |
| C70 MaasInferenceClient | ● | | | | | ● | ○ | ○ | ○ | | | | | | ○ | | ○ | | | ○ |

<!-- @end-section -->

<!-- @section: assembly-profiles -->
---

## 四、装配方案（Assembly Profiles）

框架通过 `AssemblyProfile` 描述一套组件组合，支持按名称加载预设方案。

```go
// AssemblyProfile 定义一套完整的组件装配方案。
type AssemblyProfile struct {
    Name        string
    Description string
    Required    []ComponentID   // 必须注册的组件
    Optional    []ComponentID   // 按需开启的组件
    Conflicts   []ComponentID   // 此方案中禁用的组件
}
```

### 预设方案

#### `minimal` — 最小可用集

适用于：本地开发、单机部署、快速验证

```yaml
profile: minimal
required:
  - C01  # ModelProvider（至少一个实现）
  - C03  # ProviderRegistry
  - C14  # ModelRouter
  - C30  # PricingEngine
  - C31  # BillingSession
  - C32  # FundingSource（Wallet 实现）
  - C50  # StreamProxy
  - C62  # AuditLogger
  - C70  # MaasInferenceClient（上层稳定推理端口）
optional: []
```

#### `standard` — 标准生产配置

适用于：SaaS 多租户、中等规模

```yaml
profile: standard
required:
  - C01  # ModelProvider
  - C03  # ProviderRegistry
  - C04  # ModelMapper
  - C05  # ProviderHealthMonitor
  - C06  # ConcurrencyLimiter
  - C10  # AuthGate
  - C11  # TenantContextLoader
  - C12  # RateLimiter
  - C13  # QuotaChecker
  - C14  # ModelRouter
  - C30  # PricingEngine
  - C31  # BillingSession
  - C32  # FundingSource
  - C33  # QuotaBudgetManager
  - C40  # FailoverManager
  - C41  # RetryScheduler
  - C50  # StreamProxy
  - C61  # MetricsRecorder
  - C62  # AuditLogger
  - C70  # MaasInferenceClient
optional:
  - C16  # SemanticCache
  - C34  # CircuitBreaker
  - C60  # TraceCollector
  - C63  # CostAttributor
```

#### `enterprise` — 企业全功能配置

适用于：大规模多租户、严格合规、全链路追踪

```yaml
profile: enterprise
required:
  - standard（继承所有）
  - C16  # SemanticCache
  - C17  # ContentFilter
  - C19  # TelemetryEmitter
  - C34  # CircuitBreaker
  - C60  # TraceCollector
  - C63  # CostAttributor
optional:
  - C02  # AsyncTaskProvider（视频/音频生成）
```

#### `embedded` — 嵌入式（无认证，内网信任）

适用于：作为内部 SDK 被其他服务调用，调用方自行处理认证

```yaml
profile: embedded
required:
  - C01  # ModelProvider
  - C03  # ProviderRegistry
  - C14  # ModelRouter
  - C30  # PricingEngine
  - C31  # BillingSession
  - C32  # FundingSource
  - C40  # FailoverManager
  - C41  # RetryScheduler
  - C50  # StreamProxy
  - C70  # MaasInferenceClient
conflicts:
  - C10  # 不安装 AuthGate（调用方负责）
  - C11  # 不安装 TenantContextLoader
```

<!-- @end-section -->

<!-- @section: lifecycle -->
---

## 五、组件生命周期

### 5.1 框架启动顺序

```
1. 解析装配配置（Profile + 自定义覆盖）
2. 拓扑排序：按依赖关系确定初始化顺序
3. 依赖校验：检测缺失的 depends_on
4. 按序初始化：
   L0 → L1 → L2 → L3 → L4 → L5 → L6 → L7
5. 健康自检：调用各组件的 SelfCheck()（如实现）
6. 就绪：开始接受请求
```

### 5.2 组件接口约定

所有组件**可选**实现以下框架感知接口：

```go
// ComponentLifecycle 框架感知接口（可选实现）。
// 框架通过类型断言检测，未实现则跳过对应生命周期回调。
type ComponentLifecycle interface {
    // SelfCheck 在启动完成后调用，用于验证配置有效性。
    // 返回 error 会阻止框架启动。
    SelfCheck(ctx context.Context) error

    // Shutdown 框架关闭时调用（反向拓扑顺序）。
    // ctx 含超时（默认 30s），超时后强制终止。
    Shutdown(ctx context.Context) error
}

// ComponentID 每个组件的唯一标识符。
type ComponentID string

// Component 所有组件的最小公共接口。
type Component interface {
    ID() ComponentID
}
```

### 5.3 Noop 实现规则

对于 `optional_deps` 中缺失的组件，框架注入对应的 **Noop 实现**，Noop 实现规则：

| 组件类型 | Noop 行为 |
|----------|-----------|
| `ProviderHealthMonitor` | 始终返回满分（1.0），不监控 |
| `SemanticCache` | 始终 Cache Miss，透传上游 |
| `ContentFilter` | 始终通过，不过滤 |
| `TelemetryEmitter` | 丢弃所有遥测数据 |
| `QuotaBudgetManager` | 始终允许，不限额 |
| `CircuitBreaker` | 始终关闭（不熔断） |
| `TraceCollector` | 丢弃所有 Span |
| `MetricsRecorder` | 丢弃所有指标 |
| `CostAttributor` | 不归因，只记录总量 |
| `MaasInferenceClient` | 仅测试可用：返回空文本和零 usage；生产拒绝启动 |

<!-- @end-section -->

<!-- @section: registration-api -->
---

## 六、注册 API

### 6.1 代码注册（Go）

```go
// 框架入口：构建 ComponentRegistry 并启动
func main() {
    registry := component.NewRegistry()

    // 加载预设方案
    registry.UseProfile(component.ProfileStandard)

    // 注册具体实现
    registry.Register(anthropic.NewProvider(cfg.Anthropic))
    registry.Register(openai.NewProvider(cfg.OpenAI))
    registry.Register(redis.NewConcurrencyLimiter(cfg.Redis))
    registry.Register(postgres.NewAuditLogger(cfg.DB))

    // 可选：覆盖某个组件的 Noop 实现
    registry.Register(otlp.NewTraceCollector(cfg.OTLP))

    // 校验依赖并启动（失败则 panic）
    framework := registry.Build(ctx)
    framework.Serve(":8080")
}
```

### 6.2 配置文件注册（YAML）

```yaml
# legion.yaml
profile: standard

providers:
  - type: anthropic
    id: anthropic-primary
    endpoint: https://api.anthropic.com
    credentials:
      api_key: ${ANTHROPIC_API_KEY}

  - type: openai-compatible
    id: deepseek-primary
    endpoint: https://api.deepseek.com
    credentials:
      api_key: ${DEEPSEEK_API_KEY}
    extra:
      default_model: deepseek-chat

components:
  concurrency_limiter:
    backend: redis          # "redis" | "memory"
    redis_url: ${REDIS_URL}

  audit_logger:
    backend: postgres       # "postgres" | "clickhouse" | "stdout"
    dsn: ${DATABASE_URL}

  trace_collector:
    enabled: true
    backend: otlp
    endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT}

  semantic_cache:
    enabled: false          # 显式关闭，使用 Noop
```

<!-- @end-section -->

<!-- @section: spec-index -->
---

## 七、组件规范文档索引

按优先级排列（高优先级 = 其他规范依赖它，应先写）：

| 优先级 | 组件 ID | 规范文档 | 依赖于 | 被依赖于 |
|--------|---------|----------|--------|----------|
| P0 | C00 | [component-registry-impl-spec.md](./component-registry-impl-spec.md) | — | 所有组件 |
| P1 | C01 | [model-provider-spec.md](./model-provider-spec.md) | — | C03, C50 |
| P1 | C30 | pricing-engine-spec.md | — | C31, C22 |
| P1 | C32 | funding-source-spec.md | — | C31 |
| P1 | C41 | retry-scheduler-spec.md | — | C40 |
| P2 | C02 | [async-task-provider-spec.md](./async-task-provider-spec.md) | — | — |
| P2 | C03 | provider-registry-spec.md | C01 | C14 |
| P2 | C31 | [billing-session-spec.md](./billing-session-spec.md) | C30, C32 | C63 |
| P2 | C40 | [failover-manager-spec.md](./failover-manager-spec.md) | C41 | — |
| P3 | C04 | model-mapper-spec.md | — | C14 |
| P3 | C05 | provider-health-monitor-spec.md | C01 | C14, C23, C40 |
| P3 | C14 | [model-router-spec.md](./model-router-spec.md) | C03, C04 | — |
| P3 | C33 | quota-budget-manager-spec.md | — | C13, C31 |
| P4 | C06 | concurrency-limiter-spec.md | C01 | C12 |
| P4 | C10 | auth-gate-spec.md | — | — |
| P4 | C34 | circuit-breaker-spec.md | — | C33 |
| P4 | C50 | stream-proxy-spec.md | C01 | — |
| P5 | C11 | tenant-context-loader-spec.md | — | — |
| P5 | C12 | rate-limiter-spec.md | — | — |
| P5 | C13 | quota-checker-spec.md | C33 | — |
| P5 | C16 | semantic-cache-spec.md | — | — |
| P5 | C17 | content-filter-spec.md | — | — |
| P5 | C19 | telemetry-emitter-spec.md | — | — |
| P5 | C60 | trace-collector-spec.md | — | — |
| P5 | C61 | metrics-recorder-spec.md | — | — |
| P5 | C62 | audit-logger-spec.md | — | — |
| P5 | C63 | cost-attributor-spec.md | C31 | — |
| P6 | C70 | [maas-inference-client-spec.md](./maas-inference-client-spec.md) | C01, C14 | A01, A03, A60, K13, K21, K51 |

<!-- @end-section -->
