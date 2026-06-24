---
id: "spec-component-async-task-provider-002"
title: "AsyncTaskProvider 组件规范"
aliases: ["AsyncTaskProvider规范", "异步任务提供商", "async-task-provider-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "provider", "async", "video", "audio", "image", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C02"
layer: "L1"
depends_on: []
optional_deps:
  - "C30"   # PricingEngine — 用于计费估算（可降级为提供商内置估算）
required_by: []
conflicts_with: []
assembly_profiles:
  - enterprise
---

<!-- @section: overview -->
# AsyncTaskProvider 组件规范

## 1. 组件定位

`AsyncTaskProvider` 是**长耗时生成任务**（视频生成、音频合成、图像生成、批量推理等）的提供商抽象，与 `ModelProvider`（同步推理）平级但不继承。

它解决与同步推理的根本差异：
- **同步推理**：单次 HTTP 请求，秒级返回
- **异步任务**：提交后返回 `task_id`，需轮询若干分钟到小时；计费在任务完成时才能确定

```
SubmitTask → task_id（提供商侧）
   │
   ├─ 框架持久化 task 记录 + 预扣额度
   ├─ 框架按 PollingInterval 调度 PollTask
   │       │
   │       ├─ Pending  → 继续轮询
   │       ├─ Running  → 继续轮询（可上报进度）
   │       ├─ Success  → SettleBilling → 实际结算 → 通知调用方
   │       └─ Failed   → 退还预扣 → 通知调用方
   │
   └─ 调用方通过框架查询任务状态（不直接调用 PollTask）
```

**职责边界**：
- ✅ 协议适配（提交任务、轮询、结果解析）
- ✅ 计费估算与最终结算
- ✅ HTTP 错误分类（复用 `ModelProvider` 的 `ErrorClass`）
- ❌ 不负责轮询调度（由框架的 TaskScheduler 完成）
- ❌ 不负责持久化（框架持久化 task 状态机）
- ❌ 不负责扣费（框架通过 BillingSession 协调）

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// AsyncTaskProvider 异步任务提供商。
// 实现方应是无状态的；任务状态由框架持久化。
//
// 与 ModelProvider 的关键区别：
//   - 不实现 BuildRequest/ParseResponse，因为是多步交互
//   - SubmitTask/PollTask 直接发送 HTTP 请求（接口本身约束 ctx 超时即可）
//   - 计费估算在提交前，结算在 PollTask 返回 Success 时
type AsyncTaskProvider interface {
    // ID 提供商唯一标识（与 ModelProvider 共享命名空间，不允许冲突）。
    ID() string

    // SubmitTask 提交一个异步任务。
    // 返回的 TaskSubmission 至少包含提供商侧 task_id；可携带预估完成时间。
    // 错误统一为 *ProviderError（复用 ModelProvider 错误体系）。
    SubmitTask(ctx *RequestContext) (*TaskSubmission, error)

    // PollTask 查询任务当前状态。
    // 调用方为框架的 TaskScheduler；实现方不应自行循环。
    // 返回 TaskPollResult.Status:
    //   - Pending/Running:  框架继续按 PollingInterval 轮询
    //   - Success:           Result 字段必填；框架触发 SettleBilling
    //   - Failed/Canceled:   Error 字段必填；框架退还预扣额度
    PollTask(ctx context.Context, taskID string) (*TaskPollResult, error)

    // CancelTask 取消正在进行的任务（可选实现，不支持时返回 ErrCancelUnsupported）。
    // 取消后下一次 PollTask 应返回 Canceled 状态。
    CancelTask(ctx context.Context, taskID string) error

    // EstimateBilling 在 SubmitTask 之前估算计费用量（用于预扣额度）。
    // 实现方可基于参数（时长/分辨率/采样步数）给出保守上限。
    EstimateBilling(input *ModelInput) BillingEstimate

    // SettleBilling 任务成功完成时返回最终计费用量。
    // result 来自最近一次 PollTask 的成功返回。
    // 注意：失败任务由框架直接退还预扣，不调用此方法。
    SettleBilling(result *TaskPollResult) BillingSettlement

    // PollingInterval 推荐的轮询间隔（框架会按此调度 PollTask）。
    // 可根据已运行时长返回不同值（如指数退避）。
    PollingInterval(elapsed time.Duration) time.Duration

    // ClassifyHTTPError 将 HTTP 状态码 + 响应体分类为 ErrorClass。
    // 复用 ModelProvider 的 ErrorClass 体系。
    ClassifyHTTPError(statusCode int, body []byte) ErrorClass
}
```

<!-- @end-section -->

<!-- @section: data-types -->
---

## 3. 数据类型定义

### 3.1 任务提交与轮询

```go
// TaskSubmission 是 SubmitTask 的成功返回。
type TaskSubmission struct {
    // ProviderTaskID 提供商侧的任务 ID（用于后续 PollTask）。
    ProviderTaskID string

    // EstimatedDuration 提供商预估的完成时间（可选，0 表示未知）。
    EstimatedDuration time.Duration

    // SubmittedAt 提交时间（提供商响应时间，UTC）。
    SubmittedAt time.Time

    // RawBody 上游响应原始内容（仅用于审计日志）。
    RawBody []byte
}

// TaskPollResult 是 PollTask 的返回。
type TaskPollResult struct {
    Status TaskStatus

    // Progress 0-100 整数（提供商不支持时为 0）。
    Progress int

    // Result 仅 Status=Success 时有值。
    // 包含输出资源的 URL 或字节内容。
    Result *TaskResult

    // Error 仅 Status=Failed 时有值。
    Error *ProviderError

    // Usage 实际消耗的资源量（仅 Success 时有值）。
    // 由 SettleBilling 转换为 BillingSettlement。
    Usage *TaskUsage

    // CompletedAt 完成时间（Success/Failed/Canceled 时填充）。
    CompletedAt time.Time

    // RawBody 上游响应原始内容。
    RawBody []byte
}

// TaskStatus 任务生命周期状态。
type TaskStatus int

const (
    TaskStatusPending  TaskStatus = iota // 已接收，等待开始
    TaskStatusRunning                    // 运行中
    TaskStatusSuccess                    // 成功完成
    TaskStatusFailed                     // 失败（含可重试与不可重试两类，由 Error.Class 区分）
    TaskStatusCanceled                   // 被取消
)

// TaskResult 任务输出。
type TaskResult struct {
    // Outputs 输出资源（视频/音频/图像可能多个）。
    Outputs []TaskOutput

    // Metadata 提供商返回的元数据（时长、分辨率、采样率等）。
    Metadata map[string]any
}

// TaskOutput 单个输出资源。
type TaskOutput struct {
    MediaType TaskMediaType  // video / audio / image / file
    URL       string         // 提供商签发的临时下载 URL（可能有 TTL）
    URLExpiry time.Time      // URL 过期时间（0 表示无 TTL 或永久）
    Bytes     []byte         // 直接返回的字节内容（小文件场景，与 URL 二选一）
    SizeBytes int64          // 文件大小（字节）
    MIMEType  string         // 如 "video/mp4"
}

type TaskMediaType int

const (
    MediaVideo TaskMediaType = iota
    MediaAudio
    MediaImage
    MediaFile
)
```

### 3.2 计费数据

```go
// TaskUsage 任务实际资源消耗（用于结算）。
// 各字段语义因模态不同：
//   - 视频：DurationSeconds 含义为生成视频时长
//   - 音频：DurationSeconds 含义为生成音频时长
//   - 图像：FrameCount 含义为生成图像数量
//   - 通用：StepCount 用于扩散模型采样步数
type TaskUsage struct {
    DurationSeconds float64 // 输出媒体时长（秒）
    FrameCount      int     // 输出帧数 / 图像数
    StepCount       int     // 采样/扩散步数
    PixelCount      int64   // 总像素数（分辨率 × 帧数，可选）
    InputTokens     int     // 输入提示词的 token 数（如有）
    Custom          map[string]float64 // 提供商特有的计费维度
}

// BillingEstimate 提交前的预估用量（保守上限）。
type BillingEstimate struct {
    Usage  TaskUsage
    Model  string            // 用于查 PricingEngine
    Reason string            // 估算依据描述（用于审计）
}

// BillingSettlement 任务完成后的最终用量。
type BillingSettlement struct {
    Usage  TaskUsage
    Model  string
}
```

<!-- @end-section -->

<!-- @section: lifecycle -->
---

## 4. 任务生命周期与框架协作

```
[调用方]                [框架]                          [Provider]
   │                      │                                │
   │── 提交请求 ────────►  │                                │
   │                      │── PreConsume(estimate) ─►      │  // FundingSource
   │                      │── SubmitTask ───────────────► │
   │                      │◄────── TaskSubmission ──────  │
   │                      │── 持久化 task 记录             │
   │◄── task_id ────────  │                                │
   │                      │                                │
   │                      │── (TaskScheduler 周期触发)     │
   │                      │── PollTask(taskID) ─────────► │
   │                      │◄──── TaskPollResult ─────────  │
   │                      │                                │
   │                      │   Success:                     │
   │                      │      ├─ SettleBilling ──────► │
   │                      │      │◄── BillingSettlement   │
   │                      │      ├─ Settle(actual) ──►    │  // FundingSource
   │                      │      └─ 写 cost_attribution   │
   │                      │                                │
   │                      │   Failed/Canceled:             │
   │                      │      └─ Refund(estimate)  ──► │  // FundingSource
   │                      │                                │
   │── 查询状态 ─────────►  │  (从框架持久化层读取)         │
   │◄── 当前状态 ────────  │                                │
```

### 4.1 框架职责

| 阶段 | 框架行为 |
|------|----------|
| 提交 | 调用 `EstimateBilling` → 通过 `BillingSession` 预扣 → 调用 `SubmitTask` → 持久化 task 记录 |
| 轮询 | 按 `PollingInterval(elapsed)` 调度 `PollTask`；失败按 `ClassifyHTTPError` 决定重试或切换 |
| 完成 | Success → `SettleBilling` → 实际扣费；Failed/Canceled → 退还预扣 |
| 通知 | 通过 webhook / SSE / 长轮询通知调用方（不在本规范范围内） |
| 清理 | 任务终态后保留 `RetentionDays` 天数据，之后归档 |

### 4.2 Provider 实现职责

- **无状态**：不在 Provider 实例上保存 task 状态
- **幂等 SubmitTask**：调用方传同一 `RequestContext.RequestID` 时应该返回相同的 ProviderTaskID（如提供商不支持，框架在持久化层去重）
- **轮询接口**：实现方只负责单次查询，循环调度由框架完成
- **Cancel 可选**：不支持时返回 `ErrCancelUnsupported`

<!-- @end-section -->

<!-- @section: config -->
---

## 5. 配置 Schema

```go
// AsyncTaskProviderConfig 是异步任务提供商的配置。
// 复用 ProviderConfig 的通用字段（ID/Endpoint/Credentials），扩展任务相关参数。
type AsyncTaskProviderConfig struct {
    ProviderConfig    // 嵌入通用字段

    // PollingMinInterval 轮询最小间隔（防止过度轮询）。
    PollingMinInterval time.Duration `default:"5s"  validate:"min=1s"`

    // PollingMaxInterval 轮询最大间隔（指数退避上限）。
    PollingMaxInterval time.Duration `default:"60s" validate:"max=600s"`

    // TaskTimeout 单个任务最大允许耗时（超过则框架主动 Cancel）。
    TaskTimeout time.Duration `default:"30m" validate:"min=1m,max=24h"`

    // RetentionDays 任务记录保留天数（终态后）。
    RetentionDays int `default:"30" validate:"min=1,max=365"`

    // AllowCancel 是否允许取消任务（影响调用方 API）。
    AllowCancel bool `default:"true"`
}
```

### 配置文件示例

```yaml
providers:
  - type: video-generator
    id: minimax-video-01
    endpoint: https://api.minimax.chat/v1/video_generation
    credentials:
      api_key: ${MINIMAX_API_KEY}
    polling_min_interval: 10s
    polling_max_interval: 120s
    task_timeout: 20m
    retention_days: 7
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 6. 行为契约

| 契约 | 说明 |
|------|------|
| **SubmitTask 失败不扣费** | 框架在 SubmitTask 返回错误时立即退还预扣 |
| **PollTask 幂等** | 同一 task_id 多次调用结果一致（除状态自然演进外） |
| **EstimateBilling 保守** | 估算应是上限；超估优于低估（避免欠费透支） |
| **SettleBilling 仅在 Success 调用** | 失败/取消时框架不会调用，不需要在此处理失败逻辑 |
| **PollingInterval 单调非降** | 随 elapsed 增长应不减小（指数退避或固定），便于 TaskScheduler 排程 |
| **CancelTask 异步** | Cancel 调用返回不代表任务已终止；下一次 PollTask 才能确认状态变为 Canceled |
| **ProviderTaskID 不变** | 一旦 SubmitTask 返回，ProviderTaskID 在该任务生命周期内永久有效 |
| **错误统一为 ProviderError** | 所有方法返回的非 nil error 必须是 `*ProviderError` |

<!-- @end-section -->

<!-- @section: error-handling -->
---

## 7. 错误处理

### 7.1 SubmitTask 错误

| ErrorClass | 框架处理 |
|------------|----------|
| `Transient` | 框架重试当前提供商（最多 2 次） |
| `RateLimit` | 切换到其他提供商（同模型） |
| `QuotaExhausted` | 切换提供商，触发熔断 |
| `BadRequest` | 不重试，向调用方返回错误 |
| `AuthFailure` | 告警，禁用提供商配置 |
| `Unavailable` | 切换提供商 + 熔断 |
| `ContentFiltered` | 不重试，明确告知调用方 |

### 7.2 PollTask 错误

| 场景 | 框架处理 |
|------|----------|
| 网络/临时错误 | 按 `PollingInterval(elapsed) × 2` 退避重试 |
| 404（任务不存在） | 标记 task 为 Failed（异常状态），告警 |
| 401/403 | 告警，禁用提供商；保留 task 记录 |
| 超过 `TaskTimeout` | 框架主动调用 `CancelTask`，标记 Canceled |

### 7.3 CancelTask 错误

```go
// 不支持取消时返回此错误。
var ErrCancelUnsupported = &ProviderError{
    Class:   ErrClassBadRequest,
    Code:    "cancel_unsupported",
    Message: "this provider does not support task cancellation",
}
```

<!-- @end-section -->

<!-- @section: persistence -->
---

## 8. 持久化（框架职责，规范参考）

框架使用以下表持久化任务状态（实现方不直接访问）：

```sql
-- async_tasks 框架管理的任务状态表
CREATE TABLE async_tasks (
    id                  BIGSERIAL    PRIMARY KEY,
    request_id          VARCHAR(64)  NOT NULL UNIQUE,  -- 调用方幂等键
    provider_id         VARCHAR(64)  NOT NULL,
    provider_task_id    VARCHAR(128) NOT NULL,
    user_id             VARCHAR(64)  NOT NULL,
    tenant_id           VARCHAR(64),
    model_id            VARCHAR(128) NOT NULL,
    status              VARCHAR(16)  NOT NULL,         -- pending/running/success/failed/canceled
    progress            INT          NOT NULL DEFAULT 0,
    submitted_at        TIMESTAMPTZ  NOT NULL,
    completed_at        TIMESTAMPTZ,
    next_poll_at        TIMESTAMPTZ,                   -- TaskScheduler 调度依据
    poll_count          INT          NOT NULL DEFAULT 0,
    estimated_units     BIGINT       NOT NULL,         -- 预扣配额单位
    settled_units       BIGINT,                        -- 实际消耗（Success 后填）
    result_json         JSONB,                         -- TaskResult 序列化（成功时）
    error_json          JSONB,                         -- ProviderError 序列化（失败时）
    raw_submit_body     BYTEA,
    raw_poll_body       BYTEA,
    created_at          TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (provider_id, provider_task_id)
);

CREATE INDEX idx_async_tasks_user ON async_tasks (user_id, created_at DESC);
CREATE INDEX idx_async_tasks_pending ON async_tasks (next_poll_at)
    WHERE status IN ('pending', 'running');  -- TaskScheduler 调度查询
```

<!-- @end-section -->

<!-- @section: testing -->
---

## 9. 测试规范

### 9.1 契约测试

```go
// RunAsyncTaskProviderContractTests 验证所有 AsyncTaskProvider 实现都遵守接口契约。
// 使用方式：
//   func TestMinimaxVideoProvider(t *testing.T) {
//       p := minimax.NewVideoProvider(testConfig)
//       RunAsyncTaskProviderContractTests(t, p)
//   }
func RunAsyncTaskProviderContractTests(t *testing.T, p AsyncTaskProvider) {
    t.Run("ID returns stable value", func(t *testing.T) {
        require.NotEmpty(t, p.ID())
        require.Equal(t, p.ID(), p.ID())
    })

    t.Run("PollingInterval is monotonic non-decreasing", func(t *testing.T) {
        prev := p.PollingInterval(0)
        for _, e := range []time.Duration{30*time.Second, 5*time.Minute, 1*time.Hour} {
            cur := p.PollingInterval(e)
            require.GreaterOrEqual(t, cur, prev)
            prev = cur
        }
    })

    t.Run("EstimateBilling returns non-negative usage", func(t *testing.T) {
        est := p.EstimateBilling(&ModelInput{Model: "test-model"})
        require.GreaterOrEqual(t, est.Usage.DurationSeconds, 0.0)
        require.GreaterOrEqual(t, est.Usage.FrameCount, 0)
    })

    t.Run("ClassifyHTTPError covers standard codes", func(t *testing.T) {
        require.Equal(t, ErrClassRateLimit, p.ClassifyHTTPError(429, nil))
        require.Equal(t, ErrClassAuthFailure, p.ClassifyHTTPError(401, nil))
        require.Equal(t, ErrClassBadRequest, p.ClassifyHTTPError(400, nil))
    })
}
```

### 9.2 Mock 实现

```go
// MockAsyncTaskProvider 用于测试上层组件（TaskScheduler、BillingSession）。
type MockAsyncTaskProvider struct {
    IDValue        string
    SubmitFn       func(ctx *RequestContext) (*TaskSubmission, error)
    PollFn         func(ctx context.Context, taskID string) (*TaskPollResult, error)
    CancelFn       func(ctx context.Context, taskID string) error
    EstimateFn     func(input *ModelInput) BillingEstimate
    SettleFn       func(result *TaskPollResult) BillingSettlement
    PollIntervalFn func(elapsed time.Duration) time.Duration

    SubmitCalls []*RequestContext
    PollCalls   []string
    mu          sync.Mutex
}

func (m *MockAsyncTaskProvider) ID() string { return m.IDValue }

func (m *MockAsyncTaskProvider) SubmitTask(ctx *RequestContext) (*TaskSubmission, error) {
    m.mu.Lock()
    m.SubmitCalls = append(m.SubmitCalls, ctx)
    m.mu.Unlock()
    return m.SubmitFn(ctx)
}

func (m *MockAsyncTaskProvider) PollTask(ctx context.Context, taskID string) (*TaskPollResult, error) {
    m.mu.Lock()
    m.PollCalls = append(m.PollCalls, taskID)
    m.mu.Unlock()
    return m.PollFn(ctx, taskID)
}

// ... 其他方法略
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 10. 实现检查清单

```
AsyncTaskProvider
  ☐ ID 返回稳定值
  ☐ SubmitTask 直接发送 HTTP，返回 TaskSubmission 或 *ProviderError
  ☐ PollTask 单次查询，不自循环
  ☐ CancelTask 实现或返回 ErrCancelUnsupported
  ☐ EstimateBilling 返回保守上限
  ☐ SettleBilling 仅基于成功的 PollResult
  ☐ PollingInterval 单调非降
  ☐ ClassifyHTTPError 覆盖 400/401/403/429/500/503

错误处理
  ☐ 所有错误为 *ProviderError
  ☐ ProviderError.Class 设置正确
  ☐ ProviderError.RetryAfter 在 429 时填充

并发安全
  ☐ Provider 实例无可变状态
  ☐ 多 goroutine 并发调用 SubmitTask/PollTask 安全

可选实现
  ☐ ProviderCloser（持有 HTTP 连接池时）

测试
  ☐ 通过 RunAsyncTaskProviderContractTests
  ☐ 提供商特定测试覆盖至少一种成功路径与一种失败路径
```

<!-- @end-section -->

<!-- @section: examples -->
---

## 11. 实现示例（the provider 视频生成）

```go
// the providerVideoProvider 演示性实现，仅供参考。
type the providerVideoProvider struct {
    cfg    AsyncTaskProviderConfig
    client *http.Client
}

func (p *the providerVideoProvider) ID() string { return p.cfg.ID }

func (p *the providerVideoProvider) SubmitTask(rc *RequestContext) (*TaskSubmission, error) {
    body, _ := json.Marshal(map[string]any{
        "model":  rc.Input.Model,
        "prompt": rc.Input.Messages[0].Content,
        "duration": rc.Input.ExtraParams["duration"],
    })
    req, _ := http.NewRequestWithContext(rc.Ctx, "POST",
        p.cfg.Endpoint+"/video/generate", bytes.NewReader(body))
    req.Header.Set("Authorization", "Bearer "+p.cfg.Credentials.APIKey)
    req.Header.Set("X-Request-Id", rc.RequestID)

    resp, err := p.client.Do(req)
    if err != nil {
        return nil, &ProviderError{Class: ErrClassTransient, Message: err.Error()}
    }
    defer resp.Body.Close()
    raw, _ := io.ReadAll(resp.Body)

    if resp.StatusCode >= 400 {
        return nil, &ProviderError{
            Class:   p.ClassifyHTTPError(resp.StatusCode, raw),
            Code:    fmt.Sprintf("http_%d", resp.StatusCode),
            Message: string(raw),
        }
    }

    var r struct {
        TaskID string `json:"task_id"`
    }
    json.Unmarshal(raw, &r)
    return &TaskSubmission{
        ProviderTaskID:    r.TaskID,
        SubmittedAt:       time.Now().UTC(),
        EstimatedDuration: 3 * time.Minute,
        RawBody:           raw,
    }, nil
}

func (p *the providerVideoProvider) PollingInterval(elapsed time.Duration) time.Duration {
    // 前 30s 每 5s 轮询；之后逐步退避到 60s
    switch {
    case elapsed < 30*time.Second:
        return 5 * time.Second
    case elapsed < 5*time.Minute:
        return 15 * time.Second
    default:
        return 60 * time.Second
    }
}

// ... PollTask / CancelTask / EstimateBilling / SettleBilling / ClassifyHTTPError 略
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C02 在依赖图中的位置）
- [[model-provider-spec|ModelProvider 组件规范]]（C01，姊妹组件，错误体系共享）
- pricing-engine-spec.md（C30，可选依赖，TaskUsage → 配额单位转换）
- billing-session-spec.md（C31，框架协调预扣/结算/退款）
- audit-logger-spec.md（C62，记录任务全生命周期事件）

<!-- @end-section -->
