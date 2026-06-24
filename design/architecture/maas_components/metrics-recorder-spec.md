---
id: "spec-component-metrics-recorder-061"
title: "MetricsRecorder 组件规范"
aliases: ["MetricsRecorder规范", "指标记录器", "metrics-recorder-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "observability", "metrics", "prometheus", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C61"
layer: "L7"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "C19"   # TelemetryEmitter 通过 MetricsRecorder 记录指标
assembly_profiles:
  - standard
  - enterprise
---

<!-- @section: overview -->
# MetricsRecorder 组件规范

## 1. 组件定位

`MetricsRecorder` 是框架的**指标记录出口**，接收 `TelemetryEmitter` 的指标数据，暴露给外部监控系统（Prometheus、InfluxDB、CloudWatch 等）。

**Noop 行为**：C61 未注册时，框架注入 `NoopMetricsRecorder`，丢弃所有指标。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// MetricsRecorder 记录运行时指标数据。
// 并发安全：所有方法可被多个 goroutine 并发调用。
type MetricsRecorder interface {
    // Counter 记录一个可累加的计数值（如请求总数）。
    Counter(name string, value float64, labels map[string]string)

    // Gauge 记录一个瞬时值（如当前并发数）。
    Gauge(name string, value float64, labels map[string]string)

    // Histogram 记录一个分布值（如延迟分布）。
    Histogram(name string, value float64, labels map[string]string)
}
```

<!-- @end-section -->

<!-- @section: standard-metrics -->
---

## 3. 框架标准指标

TelemetryEmitter 通过 MetricsRecorder 记录以下标准指标：

| 指标名 | 类型 | 标签 | 说明 |
|--------|------|------|------|
| `legion_request_total` | Counter | provider, model, status, strategy | 请求总数 |
| `legion_request_duration_ms` | Histogram | provider, model, status | 请求耗时（ms） |
| `legion_tokens_input_total` | Counter | provider, model | 输入 token 总量 |
| `legion_tokens_output_total` | Counter | provider, model | 输出 token 总量 |
| `legion_cost_units_total` | Counter | user, tenant | 消耗配额单位总量 |
| `legion_failover_total` | Counter | from_provider, to_provider | 故障转移次数 |
| `legion_circuit_open` | Gauge | provider | 熔断器状态（1=开，0=关）|
| `legion_concurrency_active` | Gauge | provider | 当前并发飞行请求数 |

<!-- @end-section -->

<!-- @section: implementations -->
---

## 4. 内置实现

| 实现 | 说明 |
|------|------|
| `PrometheusMetricsRecorder` | 暴露 `/metrics` 端点，Prometheus 抓取 |
| `OTLPMetricsRecorder` | 通过 OTLP 推送给 OpenTelemetry Collector |
| `NoopMetricsRecorder` | 丢弃所有指标（C61 未注册时的默认实现）|

<!-- @end-section -->

<!-- @section: config -->
---

## 5. 配置 Schema

```go
// MetricsRecorderConfig 是 MetricsRecorder 的配置。
type MetricsRecorderConfig struct {
    // Backend 指标后端。
    Backend string `default:"prometheus" validate:"oneof=prometheus otlp noop"`

    // Prometheus 配置（backend=prometheus 时）。
    Prometheus *PrometheusConfig

    // OTLP 配置（backend=otlp 时）。
    OTLP *OTLPMetricsConfig
}

type PrometheusConfig struct {
    // MetricsPath Prometheus scrape 路径（默认 /metrics）。
    MetricsPath string `default:"/metrics"`
    // Port 指标暴露端口（0 = 与主服务共用同一端口）。
    Port int `default:"0"`
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 6. 行为契约

| 契约 | 说明 |
|------|------|
| **非阻塞** | Counter/Gauge/Histogram 立即返回（原子操作）|
| **标签一致性** | 相同指标名的标签集合必须固定（Prometheus 要求）|
| **Noop 丢弃** | NoopMetricsRecorder 对所有方法为无操作 |

<!-- @end-section -->

<!-- @section: checklist -->
---

## 7. 实现检查清单

```
MetricsRecorder
  ☐ Counter / Gauge / Histogram 非阻塞（原子操作）
  ☐ 标签 key 集合固定（不动态增减标签）

PrometheusMetricsRecorder
  ☐ 注册所有框架标准指标
  ☐ /metrics 端点正确暴露

Noop 实现
  ☐ 所有方法为无操作
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C61 在依赖图中的位置）
- telemetry-emitter-spec.md（C19，主要调用方）
- trace-collector-spec.md（C60，姊妹组件）

<!-- @end-section -->
