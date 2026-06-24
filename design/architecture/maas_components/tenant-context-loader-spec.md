---
id: "spec-component-tenant-context-loader-011"
title: "TenantContextLoader 组件规范"
aliases: ["TenantContextLoader规范", "租户上下文加载器", "tenant-context-loader-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "tenant", "pipeline", "context", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C11"
layer: "L2"
depends_on: []
optional_deps: []
conflicts_with: []
required_by: []
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# TenantContextLoader 组件规范

## 1. 组件定位

`TenantContextLoader` 在 `AuthGate` 之后运行，根据 `CallerIdentity` 加载**完整的租户上下文**（计费配置、路由偏好、功能开关等），填充到 `RequestContext.TenantContext` 中，供后续所有管道节点使用。

```
AuthGate → TenantContextLoader → RateLimiter → QuotaChecker → ...
              │
              │ 加载 TenantContext
              ▼ RequestContext.TenantContext = {
                  BillingPreference, GroupRatio,
                  PinnedRoutes, FeatureFlags, ...
              }
```

**读者**：配置租户策略的运营人员、集成多租户逻辑的开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// TenantContextLoader 加载租户的完整上下文配置。
// 并发安全：同一实例可在多个 goroutine 中并发调用。
type TenantContextLoader interface {
    // Load 根据调用者身份加载租户上下文。
    // 使用内存缓存（TTL 可配置），避免每次请求查 DB。
    // 失败时返回 error，框架返回 503（配置服务不可用）。
    Load(ctx context.Context, identity *CallerIdentity) (*TenantContext, error)
}

// TenantContext 租户级配置，填充到 RequestContext 中。
type TenantContext struct {
    UserID            string
    TenantID          string
    GroupID           string

    // 计费相关
    BillingPreference BillingPreference   // subscription_first / wallet_first / ...
    GroupRatio        float64             // 计费倍率（来自 PricingEngine）

    // 路由相关
    PinnedRoutes      map[string]PinnedRoute // agentID/tenantID → 强制路由配置
    AgentRole         string               // 用于 RoleBasedStrategy
    DefaultTaskType   string               // 用于 TaskTypeStrategy

    // 功能开关
    FeatureFlags      map[string]bool      // 如 {"semantic_cache": true}

    // 配额相关
    QuotaGroupID      string              // 用于 QuotaBudgetManager 的 group 维度
}

// PinnedRoute 强制路由配置（PinnedModelStrategy 读取）。
type PinnedRoute struct {
    ProviderID string
    ModelID    string
}
```

<!-- @end-section -->

<!-- @section: config -->
---

## 3. 配置 Schema

```go
// TenantContextLoaderConfig 是 TenantContextLoader 的配置。
type TenantContextLoaderConfig struct {
    // CacheTTL 缓存租户配置的 TTL。
    CacheTTL time.Duration `default:"5m"`
    // CacheMaxSize LRU 缓存最大条目数。
    CacheMaxSize int `default:"10000"`
    // Source 配置来源（"database" | "yaml"）。
    Source string `default:"database"`
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 4. 行为契约

| 契约 | 说明 |
|------|------|
| **Load 不阻塞** | 使用缓存优先，缓存未命中时同步查 DB |
| **TenantContext 不可变** | 填入 RequestContext 后不得修改（并发请求共享读） |
| **缓存失效时降级** | DB 不可用时，若缓存有略旧数据，可使用旧数据（可配置）|

<!-- @end-section -->

<!-- @section: checklist -->
---

## 5. 实现检查清单

```
TenantContextLoader
  ☐ LRU 缓存（TTL + 大小限制）
  ☐ TenantContext 填充所有字段（BillingPreference / PinnedRoutes / FeatureFlags）
  ☐ 缓存未命中时同步查 DB
  ☐ 并发安全

测试
  ☐ 缓存命中路径
  ☐ DB 不可用时的降级行为
  ☐ 并发加载相同 tenantID 不重复查 DB（singleflight）
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C11 在依赖图中的位置）
- auth-gate-spec.md（C10，前置节点，提供 CallerIdentity）
- model-router-spec.md（C14，读取 TenantContext.PinnedRoutes）
- billing-session-spec.md（C31，读取 TenantContext.BillingPreference）

<!-- @end-section -->
