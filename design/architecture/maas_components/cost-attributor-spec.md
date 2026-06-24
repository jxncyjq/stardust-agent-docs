---
id: "spec-component-cost-attributor-063"
title: "CostAttributor 组件规范"
aliases: ["CostAttributor规范", "成本归因器", "cost-attributor-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "billing", "cost", "analytics", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C63"
layer: "L7"
depends_on:
  - "C31"   # BillingSession — 获取结算数据（实际消耗配额）
optional_deps: []
conflicts_with: []
required_by: []
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# CostAttributor 组件规范

## 1. 组件定位

`CostAttributor` 在请求结算完成后，将成本数据**按维度归因**（按 Agent、按功能、按业务标签等），为成本分析和费用分摊提供数据基础。

它与 `AuditLogger` 的区别：
- `AuditLogger`：记录每次请求的原始数据（"发生了什么"）
- `CostAttributor`：聚合成本数据，归因到业务维度（"谁花了多少钱"）

```
BillingSession.Settle() 完成
        │
        ▼
CostAttributor.Attribute(snapshot, labels)
        │
        ├── 写入 cost_attribution 表（按维度聚合）
        └── 触发成本告警（超预算时）
```

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// CostAttributor 将请求成本归因到业务维度。
// 并发安全：所有方法可被多个 goroutine 并发调用。
type CostAttributor interface {
    // Attribute 记录一次请求的成本归因（异步执行）。
    // snapshot 来自 BillingSession.Snapshot()。
    // labels 是业务标签（如 agent_id、feature、project）。
    Attribute(ctx context.Context, snapshot BillingSnapshot, labels map[string]string)
}
```

<!-- @end-section -->

<!-- @section: attribution-dimensions -->
---

## 3. 归因维度

`CostAttributor` 按以下维度聚合成本数据：

| 维度 | 说明 | 示例 |
|------|------|------|
| `user_id` | 用户级成本 | 用户自助查看用量 |
| `tenant_id` | 租户级成本 | SaaS 客户账单 |
| `agent_id` | Agent 级成本 | 哪个 Agent 最费钱 |
| `model_id` | 模型级成本 | 各模型成本对比 |
| `feature` | 功能级成本（自定义标签）| "search"、"summary" 等 |
| `provider_id` | 提供商级成本 | 各提供商费用对比 |

**数据库表（聚合）**：

```sql
-- 成本归因汇总表（按小时粒度聚合）
CREATE TABLE cost_attribution (
    id           BIGSERIAL    PRIMARY KEY,
    dimension    VARCHAR(32)  NOT NULL,  -- user_id / tenant_id / agent_id / ...
    dimension_id VARCHAR(64)  NOT NULL,  -- 维度值
    hour_bucket  TIMESTAMPTZ  NOT NULL,  -- 小时粒度
    model_id     VARCHAR(128) NOT NULL,
    provider_id  VARCHAR(64)  NOT NULL,
    request_count INT         NOT NULL DEFAULT 0,
    input_tokens  BIGINT      NOT NULL DEFAULT 0,
    output_tokens BIGINT      NOT NULL DEFAULT 0,
    cost_units   BIGINT       NOT NULL DEFAULT 0,
    updated_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (dimension, dimension_id, hour_bucket, model_id, provider_id)
);

-- 使用 INSERT ... ON CONFLICT DO UPDATE 原子累加
```

<!-- @end-section -->

<!-- @section: config -->
---

## 4. 配置 Schema

```go
// CostAttributorConfig 是 CostAttributor 的配置。
type CostAttributorConfig struct {
    // Dimensions 启用的归因维度（空 = 所有维度）。
    Dimensions []string `default:"[user_id, tenant_id, agent_id, model_id]"`

    // Granularity 聚合粒度（"hour" | "day"）。
    Granularity string `default:"hour"`

    // BudgetAlerts 成本预算告警配置。
    BudgetAlerts []BudgetAlertConfig
}

type BudgetAlertConfig struct {
    Dimension   string  // 告警维度（如 "tenant_id"）
    DimensionID string  // 具体维度值（空 = 所有）
    Period      string  // "day" | "month"
    ThresholdUnits int64 // 超过此配额单位数触发告警
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 5. 行为契约

| 契约 | 说明 |
|------|------|
| **异步执行** | Attribute 立即返回，归因写入在后台执行 |
| **写入失败静默** | DB 不可用时记录 warn 日志，不影响请求结果 |
| **原子累加** | 使用 `ON CONFLICT DO UPDATE` 保证并发写入的原子性 |
| **C63 未注册时不归因** | Noop 实现丢弃所有归因数据（仍有 AuditLogger 的原始日志）|

<!-- @end-section -->

<!-- @section: checklist -->
---

## 6. 实现检查清单

```
CostAttributor
  ☐ Attribute：异步执行，立即返回
  ☐ 写入失败静默（warn 日志）
  ☐ INSERT ON CONFLICT DO UPDATE 原子累加
  ☐ 支持多维度归因（user / tenant / agent / model / provider）
  ☐ 小时粒度聚合（可配置为日粒度）

预算告警
  ☐ 超阈值时触发外部告警（日志 WARN + 指标 + 可选 webhook）

Noop 实现
  ☐ Attribute 为无操作
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C63 在依赖图中的位置）
- billing-session-spec.md（C31，必须依赖，提供 BillingSnapshot）
- audit-logger-spec.md（C62，记录原始请求日志，与 CostAttributor 互补）
- metrics-recorder-spec.md（C61，成本指标可通过 MetricsRecorder 暴露）

<!-- @end-section -->
