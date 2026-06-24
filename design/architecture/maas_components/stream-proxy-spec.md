---
id: "spec-component-stream-proxy-050"
title: "StreamProxy 组件规范"
aliases: ["StreamProxy规范", "流式代理", "stream-proxy-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "streaming", "sse", "maas", "interface"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C50"
layer: "L6"
depends_on:
  - "C01"   # ModelProvider — 提供 ParseStreamChunk，解析提供商特有的 SSE 格式
optional_deps: []
conflicts_with: []
required_by: []
assembly_profiles:
  - minimal
  - standard
  - enterprise
  - embedded
---

<!-- @section: overview -->
# StreamProxy 组件规范

## 1. 组件定位

`StreamProxy` 处理**流式（SSE）响应**的完整生命周期：从上游 AI 服务读取 SSE 事件流，通过 `ModelProvider.ParseStreamChunk()` 解析，聚合 token 用量，并将标准化后的增量内容写回客户端。

```
上游 AI 服务（SSE）
        │ data: {...}
        │ data: {...}
        │ data: [DONE]
        ▼
  StreamProxy
  ┌──────────────────────────────────────────────────────┐
  │  Reader goroutine：读取 SSE 行                       │
  │    └── ModelProvider.ParseStreamChunk(chunk)         │ ← 提供商特有解析
  │         └── StreamEvent{Delta, Done, Error}          │
  │                                                      │
  │  Writer goroutine：写回客户端                        │
  │    └── 将 StreamEvent 转为 OpenAI 兼容 SSE 格式      │
  │                                                      │
  │  Aggregator：汇总 TokenUsage（Done 事件时）          │
  └──────────────────────────────────────────────────────┘
        │
        ▼ 客户端（HTTP SSE 响应）
```

**读者**：实现流式处理的开发者、集成 StreamProxy 的管道工程师。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

### 2.1 StreamProxy 接口

```go
// StreamProxy 处理流式响应的读取、解析和转发。
// 每次流式请求创建一个新实例（StreamProxyFactory.New()）。
// 非并发安全（单请求单实例）。
type StreamProxy interface {
    // Stream 启动流式代理：读取上游响应，解析，写回客户端。
    // upstreamBody 是上游 HTTP 响应体（io.ReadCloser）。
    // writer 是客户端响应写入器。
    // 阻塞直到流结束（Done 事件）或发生错误。
    // 返回 StreamResult（含聚合的 TokenUsage）。
    Stream(ctx context.Context, upstreamBody io.ReadCloser,
        provider ModelProvider, writer StreamWriter) (StreamResult, error)
}

// StreamProxyFactory 创建 StreamProxy 实例。
type StreamProxyFactory interface {
    New(cfg StreamProxyConfig) StreamProxy
}

// StreamWriter 是写回客户端的接口（由框架的 HTTP 层实现）。
type StreamWriter interface {
    // WriteEvent 写一个 SSE 事件到客户端响应。
    // data 是 JSON 序列化后的事件数据。
    WriteEvent(data []byte) error

    // WriteDone 写 "data: [DONE]" 终止行。
    WriteDone() error

    // Flush 强制将缓冲区内容发送给客户端。
    Flush() error
}

// StreamResult 流式处理的最终结果（Stream 返回时）。
type StreamResult struct {
    Usage       TokenUsage  // 聚合的完整 token 用量
    StopReason  StopReason  // 最终停止原因
    FullContent string      // 聚合的完整文本内容（用于审计日志）
}
```

### 2.2 事件转换规则

`StreamProxy` 将 `ModelProvider.ParseStreamChunk()` 返回的 `StreamEvent` 转换为 OpenAI 兼容格式：

```go
// StreamDelta → 写出增量内容
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion.chunk",
  "choices": [{"delta": {"content": "<delta>"}, "finish_reason": null}]
}

// StreamDone → 写出最终块 + [DONE]
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion.chunk",
  "choices": [{"delta": {}, "finish_reason": "stop"}],
  "usage": { "prompt_tokens": 100, "completion_tokens": 50, ... }
}
data: [DONE]

// StreamError → 写出错误块
{
  "error": { "message": "...", "type": "provider_error" }
}
```

<!-- @end-section -->

<!-- @section: implementation -->
---

## 3. 实现细节

### 3.1 双 goroutine 架构

参考 maas 分析中的 SSE 双 goroutine 模式：

```go
func (p *streamProxy) Stream(ctx context.Context, body io.ReadCloser,
    provider ModelProvider, writer StreamWriter) (StreamResult, error) {

    eventCh := make(chan *StreamEvent, 32)
    errCh   := make(chan error, 1)

    // Reader goroutine：解析 SSE 行
    go func() {
        defer close(eventCh)
        scanner := bufio.NewScanner(body)
        for scanner.Scan() {
            line := scanner.Bytes()
            if !bytes.HasPrefix(line, []byte("data: ")) {
                continue // 跳过注释行和空行
            }
            chunk := bytes.TrimPrefix(line, []byte("data: "))
            event, err := provider.ParseStreamChunk(chunk)
            if err != nil {
                errCh <- err
                return
            }
            if event != nil {
                eventCh <- event
            }
        }
    }()

    // Writer goroutine（主 goroutine）：聚合并写回
    var aggregator StreamAggregator
    for event := range eventCh {
        switch event.Type {
        case StreamDelta:
            aggregator.AppendDelta(event.Delta)
            writer.WriteEvent(p.formatDelta(event.Delta))
            writer.Flush()
        case StreamDone:
            aggregator.SetUsage(event.Usage)
            writer.WriteEvent(p.formatDone(event))
            writer.WriteDone()
            return aggregator.Result(), nil
        case StreamError:
            writer.WriteEvent(p.formatError(event.Error))
            return StreamResult{}, event.Error
        }
    }

    select {
    case err := <-errCh:
        return StreamResult{}, err
    default:
        return aggregator.Result(), nil
    }
}
```

### 3.2 StreamAggregator（C51，内置）

```go
// StreamAggregator 汇总流式事件，构建最终结果。
// 内置于 StreamProxy，不作为独立组件暴露。
type StreamAggregator struct {
    content strings.Builder
    usage   TokenUsage
    reason  StopReason
}

func (a *StreamAggregator) AppendDelta(delta string) {
    a.content.WriteString(delta)
}

func (a *StreamAggregator) SetUsage(u *TokenUsage) {
    if u != nil {
        a.usage = *u
    }
}

func (a *StreamAggregator) Result() StreamResult {
    return StreamResult{
        Usage:       a.usage,
        FullContent: a.content.String(),
        StopReason:  a.reason,
    }
}
```

<!-- @end-section -->

<!-- @section: config -->
---

## 4. 配置 Schema

```go
// StreamProxyConfig 是 StreamProxy 的配置。
type StreamProxyConfig struct {
    // ReadBufferSize SSE 读取缓冲区大小。
    ReadBufferSize int `default:"65536"` // 64KB

    // EventChannelSize Reader → Writer channel 的缓冲大小。
    EventChannelSize int `default:"32"`

    // FlushInterval 强制 Flush 的间隔（0 = 每个 Delta 立即 Flush）。
    // 设置间隔可降低系统调用频率，但增加客户端延迟。
    FlushInterval time.Duration `default:"0"`

    // MaxStreamDuration 流式响应的最大允许时长（防止僵尸连接）。
    MaxStreamDuration time.Duration `default:"10m"`

    // IncludeUsageInStream 是否在最终 Done 块中包含 usage 字段。
    IncludeUsageInStream bool `default:"true"`
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 5. 行为契约

| 契约 | 说明 |
|------|------|
| **StreamDone 必须携带 Usage** | 参考 ModelProvider 规范：StreamDone 事件的 Usage 不得为 nil |
| **Flush 每个 Delta** | 默认每次写入后立即 Flush，保证低延迟（可配置批量 Flush） |
| **ctx 取消时终止** | ctx 被取消时立即停止读取和写入，关闭上游连接 |
| **上游关闭时处理** | 上游连接异常关闭时，写出 StreamError 事件，不静默丢弃 |
| **FullContent 用于审计** | 聚合的完整文本写入 AuditLogger，不暴露给客户端 |
| **事件 Channel 满时 Reader 背压** | Reader goroutine 在 channel 满时阻塞（不丢弃事件）|

<!-- @end-section -->

<!-- @section: testing -->
---

## 6. 测试支持

```go
// RunStreamProxyContractTests 验证 StreamProxy 实现的行为契约。
func RunStreamProxyContractTests(t *testing.T, factory StreamProxyFactory) {
    t.Run("Stream/Normal/AggreatesUsage", ...)
    t.Run("Stream/ContextCanceled/Terminates", ...)
    t.Run("Stream/ProviderError/WritesErrorEvent", ...)
    t.Run("Stream/UpstreamClose/HandlesGracefully", ...)
    t.Run("Stream/LargeContent/NoTruncation", ...)
    t.Run("Writer/FlushAfterEachDelta", ...)
}

// MockSSEBody 构造测试用的 SSE 响应体。
func MockSSEBody(events []string) io.ReadCloser {
    var buf bytes.Buffer
    for _, e := range events {
        fmt.Fprintf(&buf, "data: %s\n\n", e)
    }
    return io.NopCloser(&buf)
}
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 7. 实现检查清单

```
StreamProxy
  ☐ 双 goroutine：Reader（解析 SSE）+ Writer（聚合写回）
  ☐ 通过 ModelProvider.ParseStreamChunk 解析（不直接解析 JSON）
  ☐ StreamDone 聚合 TokenUsage，写出最终块 + [DONE]
  ☐ StreamError 写出错误块，终止流
  ☐ ctx 取消时立即终止
  ☐ 上游连接异常关闭时写出 StreamError
  ☐ Flush 策略：默认每 Delta 立即 Flush

StreamAggregator（内置）
  ☐ AppendDelta：拼接完整文本内容
  ☐ SetUsage：处理 nil Usage（StreamDone 规范要求非 nil，但防御性处理）
  ☐ Result：返回完整快照

格式转换
  ☐ 输出 OpenAI 兼容 SSE 格式
  ☐ Done 块包含 usage（IncludeUsageInStream=true）

测试
  ☐ 运行 RunStreamProxyContractTests（全部通过）
  ☐ 大量 Delta 不截断（缓冲区溢出测试）
  ☐ ctx 取消时 goroutine 正确退出（无泄漏）
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C50 在依赖图中的位置）
- model-provider-spec.md（C01，ParseStreamChunk 接口）
- telemetry-emitter-spec.md（C19，接收 StreamResult 用于遥测）
- audit-logger-spec.md（C62，接收 FullContent 用于审计）

<!-- @end-section -->
