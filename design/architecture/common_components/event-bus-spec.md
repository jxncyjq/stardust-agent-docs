---
id: "spec-common-event-bus-X00"
title: "EventBus 组件规范"
aliases: ["EventBus规范", "事件总线", "event-bus-spec"]
type: "spec"
category: "design/architecture/common_components"
tags: ["component-spec", "common", "event-bus", "pub-sub", "streaming", "legion"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "index-common-components"

component_id: "X00"
layer: "X"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "A01"   # AgentRuntime — 流式 token 推送、LearningEvent 发布
  - "A02"   # AgentCoordinator — 任务状态广播
  - "A54"   # EvolutionEventLog — 进化事件传递
  - "A60"   # AegisReviewer — 审核结果广播
  - "A61"   # TrustScoreManager — 信任分变更广播
assembly_profiles:
  - minimal
  - standard
  - enterprise
---

<!-- @section: overview -->
# EventBus 组件规范

## 1. 组件定位

`EventBus` 是 Legion 平台的**异步发布/订阅事件总线**，是 MaaS 层与 Agent Engine 层共用的基础设施组件。

**核心用途**：

| 用途 | 发布者 | 消费者 |
|------|-------|-------|
| 流式 token 推送 | AgentRuntime（每个 LLM token delta）| Web UI / API Gateway SSE |
| 任务状态广播 | AgentCoordinator（状态流转） | 管理面板 / 用户通知 |
| LearningEvent 传递 | AgentRuntime（任务结束后）| BackgroundScheduler.gep_failure_scan |
| HardLoop 告警 | AgentCoordinator（AnomalyDetected 分支）| 管理员通知 / ApprovalService |
| 信任分变更 | TrustScoreManager | 安全审计面板 |
| 审核结果广播 | AegisReviewer | UI 实时反馈 |

```
发布者（Publisher）          EventBus              消费者（Subscriber）
AgentRuntime ──────────────────►  topic: token.delta ──────────► Web UI SSE
AgentCoordinator ──────────────►  topic: task.*     ──────────► 管理面板
AgentRuntime ──────────────────►  topic: learning.* ──────────► gep_failure_scan
TrustScoreManager ─────────────►  topic: trust.*    ──────────► 审计面板
```

**设计约束**：
- **非持久化**：EventBus 是内存 pub/sub，不持久化事件。`LearningEvent` 由 `SignalExtractor` 写入 `learning_events` 表持久化，EventBus 仅负责实时传递通知。
- **至多一次（at-most-once）**：事件丢失不影响业务正确性，仅影响实时性。
- **背压控制**：慢速消费者通过有界缓冲区和丢弃策略保护发布者不阻塞。

**读者**：实现流式推送通道的工程师、需要订阅任务状态变更的集成方。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// EventBus 是平台级异步发布/订阅事件总线。
// 并发安全：可被多个 goroutine 同时调用（内部使用 RwLock 保护订阅者表）。
// 非持久化：服务重启后所有订阅和未消费事件丢失。
type EventBus interface {
    // Publish 向指定 topic 发布事件。
    // 非阻塞：若某个订阅者的缓冲区已满，事件对该订阅者被丢弃（不阻塞发布者）。
    // topic 格式："{domain}.{event_type}"，如 "task.suspended"、"token.delta"。
    Publish(topic string, event Event) error

    // Subscribe 订阅指定 topic，返回事件接收通道。
    // bufferSize 指定该订阅者的有界缓冲区大小（默认 256）。
    // 取消订阅：关闭返回的 ctx 或调用返回的 unsubscribe 函数。
    Subscribe(ctx context.Context, topic string, bufferSize int) (<-chan Event, UnsubscribeFunc, error)

    // SubscribePattern 订阅符合 glob 模式的所有 topic（如 "task.*"、"token.*"）。
    // 仅支持单层通配符 "*"（不支持多层 "**"）。
    SubscribePattern(ctx context.Context, pattern string, bufferSize int) (<-chan Event, UnsubscribeFunc, error)

    // PublishSync 同步发布，等待所有订阅者至少尝试一次接收（或缓冲区满丢弃）。
    // 用于测试场景，生产代码优先使用 Publish。
    PublishSync(topic string, event Event) error

    // TopicStats 返回指定 topic 当前活跃订阅者数量和积压事件估算。
    // 用于可观测性指标采集，不影响事件流。
    TopicStats(topic string) TopicStats
}

// UnsubscribeFunc 取消订阅，关闭对应的事件接收通道。
type UnsubscribeFunc func()
```

<!-- @end-section -->

<!-- @section: data-types -->
---

## 3. 数据类型定义

### 3.1 事件基础结构

```go
// Event 是 EventBus 传递的事件基础结构。
type Event struct {
    ID        string          // 事件唯一 ID（UUID v4）
    Topic     string          // 事件 topic（如 "task.suspended"）
    Payload   EventPayload    // 具体事件负载（interface，由 topic 决定实际类型）
    PublishedAt time.Time     // 发布时间戳
    Source    string          // 发布者标识（如 "agent_coordinator"、"agent_runtime"）
}

// EventPayload 是事件负载的类型约束接口，所有具体负载类型都实现此接口。
type EventPayload interface {
    EventType() string  // 返回事件类型标识符
}
```

### 3.2 流式 Token 事件

```go
// TokenDeltaPayload 是 LLM 流式输出的单个 token 事件。
// topic: "token.delta"
// 发布者: AgentRuntime（§2.4 Step 3，每个 SSE token delta）
type TokenDeltaPayload struct {
    TaskID    string  // 任务 ID（用于前端路由到正确会话）
    AgentID   string  // 执行的 Agent ID
    Delta     string  // 本次 token 内容
    IsThinking bool   // 是否为 reasoning content（思考链，需特殊渲染）
    Done      bool    // 是否为流式结束标志
}

func (t *TokenDeltaPayload) EventType() string { return "token.delta" }
```

### 3.3 任务状态事件

```go
// TaskStatusPayload 是任务状态变更事件的通用结构。
// 不同 topic 使用同一结构，通过 topic 区分：
//   "task.in_progress"   — Step 3 原子锁定成功
//   "task.quality_review"— Step 7 Completed
//   "task.suspended"     — Step 7 AnomalyDetected（HardLoop）
//   "task.failed"        — Step 7 BudgetExhausted / Interrupted
//   "task.trust_blocked" — Step 5 TrustBlocked
type TaskStatusPayload struct {
    TaskID   string  // 任务 ID
    AgentID  string  // 执行的 Agent ID
    Reason   string  // 附加原因（suspended: "HardLoop: ratio={ratio}"；failed: 失败原因）
    Score    float64 // 有效信任分（trust_blocked 时有值）
}

func (t *TaskStatusPayload) EventType() string { return "task.status" }
```

### 3.4 学习事件

```go
// LearningEventPayload 是 Agent 执行结束后发布的学习信号。
// topic: "learning.event"
// 发布者: AgentRuntime（§2.4 Step 7 post_task_learning）
// 消费者: gep_failure_scan（BackgroundScheduler，§2.13）→ 触发 GepCycle
type LearningEventPayload struct {
    TaskID      string        // 任务 ID
    AgentID     string        // Agent ID
    Signal      LearningSignal // 信号类型
    IsLightweight bool        // true=仅实时传递（不写 EpisodicMemory 完整轨迹）
    Reason      string        // 原因说明（BudgetExhausted / HardLoopFailure 等）
    PublishedAt time.Time
}

// LearningSignal 是学习信号的枚举类型。
type LearningSignal int

const (
    LearningSignal_Success         LearningSignal = iota // 任务成功完成
    LearningSignal_Failure                               // 任务失败（含原因）
    LearningSignal_HardLoopFailure                       // HardLoop 触发（循环失控）
)

func (l *LearningEventPayload) EventType() string { return "learning.event" }
```

### 3.5 信任分事件

```go
// TrustScoreChangedPayload 是信任分变更事件。
// topic: "trust.score_changed"
// 发布者: TrustScoreManager（每次 logSecurityEvent 后）
type TrustScoreChangedPayload struct {
    AgentID      string  // Agent ID
    OldScore     float64 // 变更前有效分
    NewScore     float64 // 变更后有效分
    EventType    string  // 触发变更的事件类型（如 "SecretExposure"）
    Delta        float64 // 分数变化量（正/负）
    CooldownUntil *time.Time // 冷却期截止时间（nil 表示无冷却）
}

func (t *TrustScoreChangedPayload) EventType() string { return "trust.score_changed" }
```

### 3.6 统计结构

```go
// TopicStats 是 topic 的实时统计数据。
type TopicStats struct {
    Topic           string
    SubscriberCount int     // 当前活跃订阅者数
    BufferedEvents  int     // 所有订阅者缓冲区中积压的事件总数（估算）
    DroppedTotal    int64   // 自服务启动以来因缓冲区满丢弃的事件总数
}
```

<!-- @end-section -->

<!-- @section: topic-registry -->
---

## 4. Topic 注册表

Legion 平台使用的标准 topic 列表（实现时应在此注册，避免 topic 字符串散落）：

### 4.1 Agent Engine Topics

| Topic | 发布者 | 消费者 | 说明 |
|-------|-------|-------|------|
| `token.delta` | AgentRuntime | Web UI SSE | LLM 流式输出 token |
| `task.in_progress` | AgentCoordinator | 管理面板 | 任务开始执行 |
| `task.quality_review` | AgentCoordinator | 管理面板 | 任务进入质量审核 |
| `task.suspended` | AgentCoordinator | 管理员通知 | HardLoop 触发挂起 |
| `task.failed` | AgentCoordinator | 管理面板 | 任务执行失败 |
| `task.trust_blocked` | AgentCoordinator | 安全面板 | 信任分不足被阻止 |
| `learning.event` | AgentRuntime | gep_failure_scan | 任务学习信号 |
| `trust.score_changed` | TrustScoreManager | 安全审计面板 | Agent 信任分变更 |
| `quality_review.result` | AegisReviewer | Web UI / 管理面板 | Aegis 审核结果 |

### 4.2 Topic 命名规范

```
{domain}.{event_type}

domain: token / task / learning / trust / quality_review / skill / evolution
event_type: 小写，下划线分隔
```

通配符订阅示例：
- `task.*` — 订阅所有任务状态变更
- `learning.*` — 订阅所有学习事件（用于 GEP 信号采集）

<!-- @end-section -->

<!-- @section: behavior -->
---

## 5. 行为规范

### 5.1 发布语义

```
Publish(topic, event) 调用时：

  1. 查找所有订阅 topic 的订阅者（含 pattern 匹配）
  2. 对每个订阅者的缓冲通道执行 non-blocking send：
     select {
     case subscriber.ch <- event:
         // 成功投递
     default:
         // 缓冲区满，丢弃，incrementDropped()
     }
  3. 立即返回（不等待消费者处理）
```

### 5.2 订阅生命周期

```
Subscribe(ctx, topic, bufferSize) 调用时：
  1. 创建有界 channel（cap = bufferSize，默认 256）
  2. 注册到内部订阅者表（RwLock 保护）
  3. 返回 (<-chan Event, unsubscribeFn)

订阅取消（任意一种方式均可）：
  a. ctx 取消 → EventBus 内部监听 ctx.Done()，自动注销 + 关闭 channel
  b. 调用 unsubscribeFn() → 同上
  c. 订阅者 goroutine 退出 → 建议先取消 ctx 再退出，避免 channel 泄漏
```

### 5.3 背压策略

- **缓冲区大小**：默认 256，`token.delta` 建议 1024（高频 topic）
- **满缓冲丢弃**：不阻塞发布者；被丢弃的事件计入 `DroppedTotal` 指标
- **慢消费者隔离**：不同订阅者拥有独立缓冲区，慢消费者不影响其他订阅者
- **`token.delta` 特殊处理**：UI 断连后，对应订阅者通过 ctx 取消，EventBus 停止向该通道投递，避免内存积压

### 5.4 并发安全保证

- `Publish`、`Subscribe`、`Unsubscribe` 均并发安全
- 内部使用 `sync.RWMutex`：读（查找订阅者）使用读锁，写（注册/注销）使用写锁
- channel 操作在锁外进行（non-blocking send 不持锁），避免锁竞争

<!-- @end-section -->

<!-- @section: errors -->
---

## 6. 错误定义

```go
var (
    // ErrTopicEmpty topic 为空字符串。
    ErrTopicEmpty = errors.New("event_bus: topic cannot be empty")

    // ErrInvalidPattern glob 模式语法错误（如嵌套 "*" 或非法字符）。
    ErrInvalidPattern = errors.New("event_bus: invalid topic pattern")

    // ErrSubscriberClosed 订阅者 ctx 已取消，无法继续注册。
    ErrSubscriberClosed = errors.New("event_bus: subscriber context already cancelled")
)
```

<!-- @end-section -->

<!-- @section: observability -->
---

## 7. 可观测性

| 指标 | 类型 | 说明 |
|------|------|------|
| `eventbus.publish_total` | Counter | 按 topic 统计发布事件总数 |
| `eventbus.dropped_total` | Counter | 按 topic 统计因缓冲区满而丢弃的事件数 |
| `eventbus.subscriber_count` | Gauge | 按 topic 统计当前活跃订阅者数 |
| `eventbus.buffer_usage` | Histogram | 按 topic 统计订阅者缓冲区使用率（0~1） |

> 关键监控告警：`eventbus.dropped_total{topic="learning.event"}` > 0 说明 GEP 信号可能丢失，需扩大缓冲区或排查 `gep_failure_scan` 消费速度。

<!-- @end-section -->

<!-- @section: noop -->
---

## 8. Noop 实现规范

用于测试和 minimal 装配（不需要实时推送的场景）：

```go
// NoopEventBus 是 EventBus 的空操作实现。
// Publish：丢弃所有事件（不报错）。
// Subscribe：返回不会有数据的空通道（goroutine 消费时直接阻塞，直到 ctx 取消）。
type NoopEventBus struct{}

func (n *NoopEventBus) Publish(_ string, _ Event) error  { return nil }
func (n *NoopEventBus) PublishSync(_ string, _ Event) error { return nil }
func (n *NoopEventBus) Subscribe(ctx context.Context, _ string, _ int) (<-chan Event, UnsubscribeFunc, error) {
    ch := make(chan Event)
    unsubscribe := func() { close(ch) }
    go func() {
        <-ctx.Done()
        close(ch)
    }()
    return ch, unsubscribe, nil
}
func (n *NoopEventBus) SubscribePattern(ctx context.Context, _ string, bufferSize int) (<-chan Event, UnsubscribeFunc, error) {
    return n.Subscribe(ctx, "", bufferSize)
}
func (n *NoopEventBus) TopicStats(_ string) TopicStats { return TopicStats{} }
```

<!-- @end-section -->
