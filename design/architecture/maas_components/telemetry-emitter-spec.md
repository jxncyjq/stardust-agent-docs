---
id: "spec-component-telemetry-emitter-019"
title: "TelemetryEmitter 组件规范"
aliases: ["TelemetryEmitter规范", "遥测发射器", "telemetry-emitter-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "observability", "telemetry", "pipeline", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C19"
layer: "L2"
depends_on: []
optional_deps:
  - "C60"   # TraceCollector — 缺失时不发送 Span
  - "C61"   # MetricsRecorder — 缺失时不记录指标
conflicts_with: []
required_by: []
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# TelemetryEmitter 组件规范

## 1. 组件定位

`TelemetryEmitter` 是请求管道的**遥测数据汇总发射器**，在请求完成后（成功或失败）将请求的关键维度数据发送给 `TraceCollector` 和 `MetricsRecorder`。

它是遥测数据的**统一出口**，管道中其他节点不直接调用 TraceCollector/MetricsRecorder，而是通过 TelemetryEmitter 的 API 上报事件。

```
请求管道各节点 → RequestContext（积累遥测数据）
                        │
                        ▼ 请求完成
               TelemetryEmitter.Emit(ctx, result)
                    │              │
                    ▼              ▼
            TraceCollector  MetricsRecorder
            （Span 追踪）    （指标计数）
```

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// TelemetryEmitter 在请求完成时发射遥测数据。
// 非阻塞：所有发射操作异步执行，不影响响应延迟。
type TelemetryEmitter interface {
    // Emit 在请求完成时发射完整的遥测事件。
    // 异步执行，立即返回。
    Emit(ctx *RequestContext, result *RequestResult)

    // EmitEvent 发射单个业务事件（如熔断开启、配额告警）。
    // 异步执行，立即返回。
    EmitEvent(eventType string, payload map[string]any)
}

// RequestResult 请求的最终结果，用于构建遥测事件。
type RequestResult struct {
    Success     bool
    StatusCode  int
    Latency     time.Duration
    ProviderID  string
    ModelID     string
    Strategy    string          // 命中的路由策略
    Usage       TokenUsage
    Error       error
    FailoverCount int           // 故障转移次数
}
```

<!-- @end-section -->

<!-- @section: emitted-data -->
---

## 3. 发射的遥测数据

### 3.1 Span（发送给 TraceCollector）

```
Span: legion.request
  attributes:
    provider_id, model_id, route_strategy
    user_id, tenant_id
    input_tokens, output_tokens
    failover_count, status_code
  duration: 请求总耗时
  status: OK / ERROR
```

### 3.2 指标（发送给 MetricsRecorder）

| 指标名 | 类型 | 说明 |
|--------|------|------|
| `legion.request.total` | Counter | 总请求数（含 provider、status 标签） |
| `legion.request.duration` | Histogram | 请求延迟分布 |
| `legion.tokens.input` | Counter | 输入 token 总量 |
| `legion.tokens.output` | Counter | 输出 token 总量 |
| `legion.failover.count` | Counter | 故障转移次数 |
| `legion.error.total` | Counter | 错误数（含 error_type 标签） |

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 4. 行为契约

| 契约 | 说明 |
|------|------|
| **非阻塞** | Emit 立即返回，遥测数据异步发送 |
| **失败静默** | TraceCollector / MetricsRecorder 发送失败不影响请求结果 |
| **C60/C61 均缺失时为 Noop** | 两者均未注册时，Emit 为无操作 |

<!-- @end-section -->

<!-- @section: checklist -->
---

## 5. 实现检查清单

```
TelemetryEmitter
  ☐ Emit：非阻塞，异步执行
  ☐ TraceCollector 缺失时跳过 Span 发送
  ☐ MetricsRecorder 缺失时跳过指标记录
  ☐ 发送失败静默（记录 warn 日志，不向上传播错误）
  ☐ 并发安全
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C19 在依赖图中的位置）
- trace-collector-spec.md（C60，可选依赖）
- metrics-recorder-spec.md（C61，可选依赖）

<!-- @end-section -->
