---
id: "spec-component-trace-collector-060"
title: "TraceCollector 组件规范"
aliases: ["TraceCollector规范", "链路追踪", "trace-collector-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "observability", "tracing", "opentelemetry", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C60"
layer: "L7"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "C19"   # TelemetryEmitter 通过 TraceCollector 发送 Span
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# TraceCollector 组件规范

## 1. 组件定位

`TraceCollector` 是框架的**分布式追踪出口**，接收 `TelemetryEmitter` 发送的 Span 数据，并转发给外部追踪系统（Jaeger、Zipkin、OTLP 等）。

它不参与请求处理逻辑，是纯粹的可观测性组件。

**Noop 行为**：C60 未注册时，框架注入 `NoopTraceCollector`，丢弃所有 Span。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// TraceCollector 接收并转发分布式追踪 Span。
// 并发安全：所有方法可被多个 goroutine 并发调用。
type TraceCollector interface {
    // StartSpan 开始一个新的 Span。
    // 返回 Span 对象，调用方在完成时调用 span.End()。
    StartSpan(ctx context.Context, operationName string,
        attrs map[string]any) (context.Context, Span)
}

// Span 代表一个追踪区间。
type Span interface {
    // SetAttributes 设置 Span 属性（键值对）。
    SetAttributes(attrs map[string]any)
    // SetStatus 设置 Span 状态（OK / ERROR）。
    SetStatus(code SpanStatusCode, message string)
    // End 结束 Span（记录结束时间并发送）。
    End()
}

type SpanStatusCode int
const (
    SpanStatusOK    SpanStatusCode = iota
    SpanStatusError
)
```

<!-- @end-section -->

<!-- @section: implementations -->
---

## 3. 内置实现

| 实现 | 说明 |
|------|------|
| `OTLPTraceCollector` | 通过 OTLP gRPC/HTTP 协议发送到 OpenTelemetry Collector |
| `JaegerTraceCollector` | 直接发送到 Jaeger |
| `NoopTraceCollector` | 丢弃所有 Span（C60 未注册时的默认实现）|

```go
// OTLPTraceCollector 使用 OpenTelemetry SDK 实现。
// 推荐用于生产（可对接 Jaeger、Tempo、Datadog 等任意后端）。
type OTLPTraceCollector struct {
    exporter otlptrace.Exporter
    provider *sdktrace.TracerProvider
}
```

<!-- @end-section -->

<!-- @section: config -->
---

## 4. 配置 Schema

```go
// TraceCollectorConfig 是 TraceCollector 的配置。
type TraceCollectorConfig struct {
    // Backend 追踪后端。
    Backend string `default:"otlp" validate:"oneof=otlp jaeger noop"`

    // Endpoint OTLP/Jaeger 端点。
    Endpoint string

    // SampleRate 采样率 [0, 1]（1.0 = 全量采样）。
    SampleRate float64 `default:"0.1" validate:"min=0,max=1"`

    // ServiceName 在追踪系统中显示的服务名。
    ServiceName string `default:"legion"`
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 5. 行为契约

| 契约 | 说明 |
|------|------|
| **非阻塞发送** | Span 数据异步批量发送，不阻塞请求路径 |
| **发送失败静默** | 追踪后端不可用时记录 warn 日志，不影响请求 |
| **Noop 丢弃** | NoopTraceCollector 对所有方法为无操作 |

<!-- @end-section -->

<!-- @section: checklist -->
---

## 6. 实现检查清单

```
TraceCollector
  ☐ StartSpan：非阻塞，Span 数据批量异步发送
  ☐ 发送失败静默（warn 日志）
  ☐ SampleRate 控制采样

Noop 实现
  ☐ StartSpan 返回 NoopSpan（所有方法为无操作）
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C60 在依赖图中的位置）
- telemetry-emitter-spec.md（C19，主要调用方）
- metrics-recorder-spec.md（C61，姊妹组件）

<!-- @end-section -->
