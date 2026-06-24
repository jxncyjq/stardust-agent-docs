---
id: "plans-agent-p11-platform-integration-001"
title: "Agent P11 平台集成就绪计划"
type: "implementation-plan"
category: "plans/agent"
tags: ["plan", "agent", "p11", "platform-integration", "openapi", "observability", "security", "ops"]
version: "0.1.0"
created: "2026-05-15"
updated: "2026-05-15"
status: "draft"
related_docs:
  - path: "../../design/architecture/agent_components/index.md"
    relation: "derived_from"
  - path: "../../design/architecture/agent_components/agent-component-registry.md"
    relation: "derived_from"
  - path: "../../design/architecture/common_components/index.md"
    relation: "references"
  - path: "./p10-component-parity-plan.md"
    relation: "follows"
---

# Agent P11 Platform Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 P10 已完成的 Agent 组件对齐能力推进到平台集成就绪状态，重点补齐 API 契约、事件流、租户边界、外部观测和数据运维。

**Architecture:** P11 不改变 A00-A70 的核心组件职责，而是在组件外侧增加稳定平台契约。HTTP/API 契约由 `internal/server` 与 `internal/compat` 守护，事件流围绕 X00 EventBus 落到 SSE/订阅接口，安全边界围绕 company/tenant 与 X02 审计链增强，数据运维围绕 SQLite schema、retention 和 export 做可回滚扩展。

**Tech Stack:** Go 1.26.0、net/http、SQLite、PowerShell、GitHub Actions、JSON OpenAPI 3.1、server-sent events。

---

## P11 定位

P10 已完成组件级 parity：A03、A20-A23、A30-A32、A52-A54、A60-A64、A70 都具备测试守护。P11 的定位是 **platform integration readiness**：让这些能力可以被前端、控制台、外部平台服务和运维系统稳定消费。

P11 不做这些事：
- 不引入真实外部 Prometheus/OTel 后端，仅提供可对接出口与兼容测试。
- 不实现完整跨租户 RBAC 产品形态，只补齐 Agent 服务层必须具备的 tenant/company 边界。
- 不重写 SQLite 迁移框架，只在现有 schema 版本机制上增加 retention/export/import 的最小闭环。

## 设计依据

| 设计来源 | P11 关注点 |
|----------|------------|
| `agent_components/index.md` | A01/A02/A60-A64/A70 的外部可观测、审核、任务与工作流接口 |
| `agent-component-registry.md` | X00/X02/X04/X05/C70 作为稳定端口，避免直接依赖 provider 细节 |
| `common_components/index.md` | EventBus、ImmutableAuditLog、PathGuard、OutputSanitizer 的跨模块共用边界 |
| P10 component parity | 已完成组件行为，P11 只增加平台化契约和运维面 |

## 文件结构规划

| 路径 | 动作 | 职责 |
|------|------|------|
| `legion/legionAgent/internal/server/openapi.go` | Create | 生成 Agent HTTP API 的 OpenAPI 3.1 JSON 文档 |
| `legion/legionAgent/internal/server/openapi_test.go` | Create | 验证 OpenAPI 包含 task/workflow/quality/diagnostics/event schema |
| `legion/legionAgent/internal/compat/openapi_golden_test.go` | Create | OpenAPI 兼容性 golden 测试，阻止破坏性字段变更 |
| `legion/legionAgent/internal/compat/testdata/openapi-agent.json` | Create | Agent API 契约 golden |
| `legion/legionAgent/internal/observability/eventbus.go` | Create/Modify | X00 EventBus 的内存发布订阅、缓冲、关闭语义 |
| `legion/legionAgent/internal/server/events.go` | Create/Modify | `/v1/events` SSE 订阅，支持 task/workflow/quality 事件过滤 |
| `legion/legionAgent/internal/server/events_test.go` | Create | 验证 SSE 格式、过滤、断开清理和敏感信息净化 |
| `legion/legionAgent/internal/security/tenant.go` | Create | company/tenant 作用域、请求身份、资源访问判定 |
| `legion/legionAgent/internal/server/authz.go` | Create/Modify | HTTP 层租户鉴权中间件和资源 company 校验 |
| `legion/legionAgent/internal/server/authz_test.go` | Create | 验证跨 company task/workflow/quality 访问被拒绝 |
| `legion/legionAgent/internal/observability/prometheus.go` | Create | 将现有 MetricsSnapshot 导出为 Prometheus text format |
| `legion/legionAgent/internal/server/metrics_test.go` | Modify | 验证 `/metrics` 支持 JSON 与 Prometheus text 两种格式 |
| `legion/legionAgent/internal/storage/retention.go` | Create | 审计、运行事件、质量历史的数据保留与归档接口 |
| `legion/legionAgent/internal/storage/retention_test.go` | Create | 验证 retention 不删除未过期数据、归档摘要可审计 |
| `legion/legionAgent/internal/cli/command.go` | Modify | `agent data export` / `agent data retention` 命令 |
| `docs/agents/legion-agent/openapi.md` | Create | HTTP API 契约、兼容性和变更规则 |
| `docs/agents/legion-agent/events.md` | Create | EventBus/SSE 事件名、payload 摘要与脱敏规则 |
| `docs/agents/legion-agent/security-tenancy.md` | Create | tenant/company 边界与运维配置说明 |
| `docs/agents/legion-agent/data-retention.md` | Create | SQLite 数据保留、归档与导出操作说明 |

## P11 任务清单

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P11-001 | P0 | API/X05/Compat | OpenAPI 3.1 契约与兼容性门禁 | `done` | `internal/server`, `internal/compat`, `docs/agents/legion-agent` | `/openapi.json` 可用，golden 防破坏，schema 不暴露 secret/prompt |
| AG-P11-002 | P0 | X00/A01/A02/A60-A64/A70 | EventBus 与 SSE 平台事件流 | `done` | `internal/observability`, `internal/server` | task/workflow/quality 事件可订阅、可过滤、断开可清理 |
| AG-P11-003 | P0 | A02/A10/A70/X02 | Tenant/company 安全边界 | `done` | `internal/security`, `internal/server`, `internal/storage` | 跨 company 访问 task/workflow 被拒绝并审计；quality/audit 查询 API 待新增时复用同一边界 |
| AG-P11-004 | P1 | Observability/X00/X02 | Prometheus text exporter 与外部观测字段稳定化 | `done` | `internal/observability`, `internal/server`, `docs/agents/legion-agent` | `/metrics?format=prometheus` 输出稳定，字段无 prompt/secret |
| AG-P11-005 | P1 | Storage/X02/A60-A64 | 数据保留、质量历史归档与导出 | `done` | `internal/storage`, `internal/cli`, `docs/agents/legion-agent` | retention dry-run/apply 可测，归档摘要写 audit |
| AG-P11-006 | P1 | CI/Docs | P11 总验收与文档索引同步 | `done` | `.github/workflows`, `docs/agents/legion-agent`, `docs/plans/03-agent` | openapi/event/security/retention/test/vet/build/smoke 全部通过 |

---

## Task 1: OpenAPI 3.1 契约与兼容性门禁

**Files:**
- Create: `legion/legionAgent/internal/server/openapi.go`
- Create: `legion/legionAgent/internal/server/openapi_test.go`
- Create: `legion/legionAgent/internal/compat/openapi_golden_test.go`
- Create: `legion/legionAgent/internal/compat/testdata/openapi-agent.json`
- Create: `docs/agents/legion-agent/openapi.md`
- Modify: `legion/legionAgent/internal/server/http.go`
- Modify: `docs/agents/legion-agent/index.md`

- [x] **Step 1: 写失败测试**

在 `internal/server/openapi_test.go` 添加：

```go
func TestOpenAPISpecIncludesCorePlatformRoutes(t *testing.T) {
	spec := BuildOpenAPISpec()
	requiredPaths := []string{
		"/healthz",
		"/readyz",
		"/metrics",
		"/diagnostics",
		"/v1/tasks",
		"/v1/tasks/{id}",
		"/v1/workflows",
		"/v1/workflows/{id}",
		"/v1/workflows/{id}/events",
		"/v1/events",
	}
	for _, path := range requiredPaths {
		if _, ok := spec.Paths[path]; !ok {
			t.Errorf("BuildOpenAPISpec().Paths missing %q", path)
		}
	}
	if _, ok := spec.Components.Schemas["DiagnosticsSnapshot"]; !ok {
		t.Errorf("BuildOpenAPISpec().Components.Schemas missing DiagnosticsSnapshot")
	}
}
```

Run:

```powershell
go test ./internal/server -run TestOpenAPISpecIncludesCorePlatformRoutes -count=1
```

Expected: FAIL，`BuildOpenAPISpec` 尚未定义。

- [x] **Step 2: 实现最小 OpenAPI 类型**

在 `internal/server/openapi.go` 定义：

```go
package server

type OpenAPISpec struct {
	OpenAPI    string                    `json:"openapi"`
	Info       OpenAPIInfo               `json:"info"`
	Paths      map[string]OpenAPIPathItem `json:"paths"`
	Components OpenAPIComponents         `json:"components"`
}

type OpenAPIInfo struct {
	Title   string `json:"title"`
	Version string `json:"version"`
}

type OpenAPIPathItem struct {
	Get  *OpenAPIOperation `json:"get,omitempty"`
	Post *OpenAPIOperation `json:"post,omitempty"`
}

type OpenAPIOperation struct {
	OperationID string              `json:"operationId"`
	Summary     string              `json:"summary"`
	Responses   map[string]any      `json:"responses"`
	Security    []map[string][]string `json:"security,omitempty"`
}

type OpenAPIComponents struct {
	Schemas         map[string]any `json:"schemas"`
	SecuritySchemes map[string]any `json:"securitySchemes"`
}
```

- [x] **Step 3: 实现 `BuildOpenAPISpec`**

在 `internal/server/openapi.go` 添加：

```go
func BuildOpenAPISpec() OpenAPISpec {
	return OpenAPISpec{
		OpenAPI: "3.1.0",
		Info: OpenAPIInfo{Title: "Legion Agent API", Version: "0.1.0"},
		Paths: map[string]OpenAPIPathItem{
			"/healthz": {Get: operation("getHealthz", "Health check", false)},
			"/readyz": {Get: operation("getReadyz", "Readiness check", false)},
			"/metrics": {Get: operation("getMetrics", "Metrics snapshot", true)},
			"/diagnostics": {Get: operation("getDiagnostics", "Diagnostics snapshot", true)},
			"/openapi.json": {Get: operation("getOpenAPI", "OpenAPI contract", false)},
			"/v1/tasks": {Post: operation("submitTask", "Submit task", true)},
			"/v1/tasks/{id}": {Get: operation("getTask", "Get task status", true)},
			"/v1/workflows": {Post: operation("submitWorkflow", "Submit workflow", true)},
			"/v1/workflows/{id}": {Get: operation("getWorkflow", "Get workflow state", true)},
			"/v1/workflows/{id}/events": {Post: operation("resumeWorkflowEvent", "Resume workflow event", true)},
			"/v1/events": {Get: operation("subscribeEvents", "Subscribe platform events", true)},
		},
		Components: OpenAPIComponents{
			Schemas: map[string]any{
				"TaskSubmitRequest": map[string]any{"type": "object"},
				"WorkflowSubmitRequest": map[string]any{"type": "object"},
				"DiagnosticsSnapshot": map[string]any{"type": "object"},
				"MetricsSnapshot": map[string]any{"type": "object"},
				"EventEnvelope": map[string]any{"type": "object"},
			},
			SecuritySchemes: map[string]any{
				"AdminToken": map[string]any{"type": "apiKey", "in": "header", "name": "Authorization"},
			},
		},
	}
}

func operation(id string, summary string, secured bool) *OpenAPIOperation {
	op := &OpenAPIOperation{
		OperationID: id,
		Summary:     summary,
		Responses:   map[string]any{"200": map[string]any{"description": "OK"}},
	}
	if secured {
		op.Security = []map[string][]string{{"AdminToken": {}}}
	}
	return op
}
```

- [x] **Step 4: 增加 `/openapi.json` 路由**

在 `internal/server/http.go` 的路由注册处增加：

```go
mux.HandleFunc("GET /openapi.json", func(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, BuildOpenAPISpec())
})
```

若当前 Go 版本或路由写法不使用 method pattern，则按既有 `http.go` 的路由风格注册同等路径。

- [x] **Step 5: 增加 golden 测试**

在 `internal/compat/openapi_golden_test.go` 添加：

```go
func TestOpenAPIGolden(t *testing.T) {
	got, err := json.MarshalIndent(server.BuildOpenAPISpec(), "", "  ")
	if err != nil {
		t.Fatalf("marshal OpenAPI spec: %v", err)
	}
	got = append(got, '\n')
	want, err := os.ReadFile("testdata/openapi-agent.json")
	if err != nil {
		t.Fatalf("read OpenAPI golden: %v", err)
	}
	if diff := cmp.Diff(string(want), string(got)); diff != "" {
		t.Errorf("OpenAPI golden mismatch (-want +got):\n%s", diff)
	}
}
```

Run:

```powershell
go test ./internal/compat -run TestOpenAPIGolden -count=1
```

Expected: FAIL，golden 文件尚未生成。

- [x] **Step 6: 生成 golden 并验证**

用测试输出或小工具写入 `internal/compat/testdata/openapi-agent.json`，然后运行：

```powershell
go test ./internal/server ./internal/compat -run "TestOpenAPISpecIncludesCorePlatformRoutes|TestOpenAPIGolden" -count=1
```

Expected: PASS。

---

## Task 2: EventBus 与 SSE 平台事件流

**Files:**
- Create/Modify: `legion/legionAgent/internal/observability/eventbus.go`
- Create/Modify: `legion/legionAgent/internal/server/events.go`
- Create: `legion/legionAgent/internal/server/events_test.go`
- Create: `docs/agents/legion-agent/events.md`
- Modify: `legion/legionAgent/internal/server/http.go`

- [x] **Step 1: 写失败测试**

在 `internal/server/events_test.go` 添加：

```go
func TestSSEEventsFiltersAndSanitizesPayload(t *testing.T) {
	bus := observability.NewEventBus(8)
	srv := NewHTTPServer(HTTPServerConfig{
		AdminToken: "token",
		EventBus:   bus,
	})
	req := httptest.NewRequest(http.MethodGet, "/v1/events?type=task.completed", nil)
	req.Header.Set("Authorization", "Bearer token")
	rec := httptest.NewRecorder()
	go func() {
		_ = bus.Publish(context.Background(), observability.EventEnvelope{
			Type: "task.completed",
			Data: map[string]any{"task_id": "task-1", "prompt": "secret prompt"},
		})
		_ = bus.Close()
	}()
	srv.ServeHTTP(rec, req)
	body := rec.Body.String()
	if !strings.Contains(body, "event: task.completed") {
		t.Errorf("SSE body = %q, want task.completed event", body)
	}
	if strings.Contains(body, "secret prompt") {
		t.Errorf("SSE body leaked prompt: %q", body)
	}
}
```

Run:

```powershell
go test ./internal/server -run TestSSEEventsFiltersAndSanitizesPayload -count=1
```

Expected: FAIL，`EventBus` 或 `/v1/events` 尚未实现。

- [x] **Step 2: 实现事件信封与订阅**

在 `internal/observability/eventbus.go` 定义：

```go
type EventEnvelope struct {
	ID        string         `json:"id"`
	Type      string         `json:"type"`
	SubjectID string         `json:"subject_id,omitempty"`
	Data      map[string]any `json:"data"`
	CreatedAt time.Time      `json:"created_at"`
}

type EventBus struct {
	mu          sync.Mutex
	buffer      int
	closed      bool
	subscribers map[chan EventEnvelope]struct{}
}
```

实现：
- `NewEventBus(buffer int) *EventBus`
- `Publish(ctx context.Context, event EventEnvelope) error`
- `Subscribe(ctx context.Context) (<-chan EventEnvelope, func())`
- `Close() error`

- [x] **Step 3: 实现 SSE 写出**

在 `internal/server/events.go` 添加：

```go
func writeSSEEvent(w io.Writer, event observability.EventEnvelope) error {
	data, err := json.Marshal(sanitizeEventData(event.Data))
	if err != nil {
		return err
	}
	if _, err := fmt.Fprintf(w, "event: %s\n", event.Type); err != nil {
		return err
	}
	if event.ID != "" {
		if _, err := fmt.Fprintf(w, "id: %s\n", event.ID); err != nil {
			return err
		}
	}
	_, err = fmt.Fprintf(w, "data: %s\n\n", data)
	return err
}
```

`sanitizeEventData` 必须删除 `prompt`、`input`、`secret`、`api_key` 字段。

- [x] **Step 4: 注册 `/v1/events`**

路由行为：
- 需要 admin token。
- 查询参数 `type=task.completed` 时只返回同类型事件。
- 客户端断开时取消订阅。
- `Content-Type` 为 `text/event-stream`。

Run:

```powershell
go test ./internal/server -run TestSSEEventsFiltersAndSanitizesPayload -count=1
```

Expected: PASS。

---

## Task 3: Tenant/company 安全边界

**Files:**
- Create: `legion/legionAgent/internal/security/tenant.go`
- Create/Modify: `legion/legionAgent/internal/server/authz.go`
- Create: `legion/legionAgent/internal/server/authz_test.go`
- Modify: `legion/legionAgent/internal/storage/sqlite.go`
- Create: `docs/agents/legion-agent/security-tenancy.md`

- [x] **Step 1: 写失败测试**

在 `internal/server/authz_test.go` 添加：

```go
func TestHTTPRejectsCrossCompanyTaskAccess(t *testing.T) {
	repo := openTestSQLiteRepository(t)
	saveTestTask(t, repo, domain.Task{
		ID:        "task-1",
		CompanyID: "company-a",
		AgentID:   "agent-1",
		Status:    domain.TaskStatusPending,
		CreatedAt: time.Now(),
	})
	srv := NewHTTPServer(HTTPServerConfig{
		AdminToken: "token",
		Repository: repo,
	})
	req := httptest.NewRequest(http.MethodGet, "/v1/tasks/task-1", nil)
	req.Header.Set("Authorization", "Bearer token")
	req.Header.Set("X-Company-ID", "company-b")
	rec := httptest.NewRecorder()
	srv.ServeHTTP(rec, req)
	if rec.Code != http.StatusForbidden {
		t.Fatalf("GET /v1/tasks/task-1 status = %d, want %d", rec.Code, http.StatusForbidden)
	}
}
```

Run:

```powershell
go test ./internal/server -run TestHTTPRejectsCrossCompanyTaskAccess -count=1
```

Expected: FAIL，company 作用域尚未被 HTTP 层强制检查。

- [x] **Step 2: 定义 tenant 上下文**

在 `internal/security/tenant.go` 添加：

```go
package security

import "net/http"

type Principal struct {
	CompanyID string
	SubjectID string
	Role      string
}

func PrincipalFromRequest(r *http.Request) Principal {
	return Principal{
		CompanyID: r.Header.Get("X-Company-ID"),
		SubjectID: r.Header.Get("X-Subject-ID"),
		Role:      r.Header.Get("X-Role"),
	}
}

func CanAccessCompany(principal Principal, companyID string) bool {
	if principal.CompanyID == "" {
		return false
	}
	return principal.CompanyID == companyID
}
```

- [x] **Step 3: HTTP 资源鉴权**

在 `internal/server/authz.go` 添加：

```go
func requireCompanyAccess(w http.ResponseWriter, r *http.Request, companyID string) bool {
	principal := security.PrincipalFromRequest(r)
	if !security.CanAccessCompany(principal, companyID) {
		writeError(w, http.StatusForbidden, "company access denied")
		return false
	}
	return true
}
```

在 task/workflow/quality/audit 查询入口中读取资源 company_id 后调用该函数。

- [x] **Step 4: 记录拒绝审计**

跨 company 被拒绝时写入 audit event：

```go
domain.AuditEvent{
	ID:          newID(),
	RequestID:   requestIDFromContext(r.Context()),
	SubjectType: "company",
	SubjectID:   principal.CompanyID,
	Action:      "access_denied.cross_company",
	Hash:        hashString(resourceID),
	CreatedAt:   time.Now(),
}
```

Run:

```powershell
go test ./internal/server ./internal/storage -run "TestHTTPRejectsCrossCompanyTaskAccess|TestSQLiteAuditEvents" -count=1
```

Expected: PASS。

---

## Task 4: Prometheus text exporter 与外部观测字段稳定化

**Files:**
- Create: `legion/legionAgent/internal/observability/prometheus.go`
- Modify: `legion/legionAgent/internal/server/http.go`
- Modify: `legion/legionAgent/internal/server/metrics_test.go`
- Create: `docs/agents/legion-agent/observability.md`

- [x] **Step 1: 写失败测试**

在 `internal/server/metrics_test.go` 增加：

```go
func TestMetricsPrometheusFormat(t *testing.T) {
	recorder := observability.NewMetricsRecorder(time.Now)
	recorder.RecordHTTP("GET", "/readyz", 200, 10*time.Millisecond)
	srv := NewHTTPServer(HTTPServerConfig{
		AdminToken: "token",
		Metrics:    recorder,
	})
	req := httptest.NewRequest(http.MethodGet, "/metrics?format=prometheus", nil)
	req.Header.Set("Authorization", "Bearer token")
	rec := httptest.NewRecorder()
	srv.ServeHTTP(rec, req)
	body := rec.Body.String()
	if !strings.Contains(body, "legion_agent_http_requests_total") {
		t.Errorf("Prometheus metrics body = %q, want http counter", body)
	}
	if strings.Contains(body, "prompt") || strings.Contains(body, "secret") {
		t.Errorf("Prometheus metrics leaked sensitive field: %q", body)
	}
}
```

Run:

```powershell
go test ./internal/server -run TestMetricsPrometheusFormat -count=1
```

Expected: FAIL，Prometheus text format 尚未实现。

- [x] **Step 2: 实现 text exporter**

在 `internal/observability/prometheus.go` 添加：

```go
func PrometheusText(snapshot MetricsSnapshot) string {
	var b strings.Builder
	writeMetric(&b, "legion_agent_http_requests_total", float64(snapshot.HTTP.RequestsTotal))
	writeMetric(&b, "legion_agent_tasks_started_total", float64(snapshot.Tasks.StartedTotal))
	writeMetric(&b, "legion_agent_tasks_completed_total", float64(snapshot.Tasks.CompletedTotal))
	writeMetric(&b, "legion_agent_model_calls_total", float64(snapshot.Model.CallsTotal))
	writeMetric(&b, "legion_agent_workflows_completed_total", float64(snapshot.Workflows.CompletedTotal))
	return b.String()
}

func writeMetric(b *strings.Builder, name string, value float64) {
	fmt.Fprintf(b, "# TYPE %s counter\n%s %.0f\n", name, name, value)
}
```

字段名必须只包含聚合数值，不拼接 task prompt、tool input、secret、api key。

- [x] **Step 3: `/metrics` 支持格式协商**

行为：
- 默认返回现有 JSON。
- `?format=prometheus` 返回 `text/plain; version=0.0.4`。
- admin token 规则保持不变。

Run:

```powershell
go test ./internal/server -run TestMetricsPrometheusFormat -count=1
```

Expected: PASS。

---

## Task 5: 数据保留、质量历史归档与导出

**Files:**
- Create: `legion/legionAgent/internal/storage/retention.go`
- Create: `legion/legionAgent/internal/storage/retention_test.go`
- Modify: `legion/legionAgent/internal/cli/command.go`
- Create: `docs/agents/legion-agent/data-retention.md`

- [x] **Step 1: 写失败测试**

在 `internal/storage/retention_test.go` 添加：

```go
func TestRetentionPlanDryRunDoesNotDeleteRecentQualityHistory(t *testing.T) {
	ctx := context.Background()
	repo := openTestSQLiteRepository(t)
	now := time.Date(2026, 5, 15, 12, 0, 0, 0, time.UTC)
	if err := repo.AppendQualityEvalRun(ctx, quality.EvalRunRecord{
		ID: "eval-new", AgentID: "agent-1", TaskID: "task-1", Component: "planner",
		Status: quality.EvalNormal, Score: 1, CreatedAt: now.Add(-24 * time.Hour),
	}); err != nil {
		t.Fatalf("AppendQualityEvalRun(new) error = %v", err)
	}
	plan, err := repo.PlanRetention(ctx, RetentionPolicy{
		Now: now,
		QualityHistoryMaxAge: 7 * 24 * time.Hour,
		DryRun: true,
	})
	if err != nil {
		t.Fatalf("PlanRetention() error = %v", err)
	}
	if plan.QualityHistoryDeleted != 0 {
		t.Errorf("PlanRetention().QualityHistoryDeleted = %d, want 0", plan.QualityHistoryDeleted)
	}
}
```

Run:

```powershell
go test ./internal/storage -run TestRetentionPlanDryRunDoesNotDeleteRecentQualityHistory -count=1
```

Expected: FAIL，retention API 尚未实现。

- [x] **Step 2: 定义 retention 类型**

在 `internal/storage/retention.go` 添加：

```go
type RetentionPolicy struct {
	Now                  time.Time
	AuditMaxAge           time.Duration
	RuntimeEventMaxAge    time.Duration
	QualityHistoryMaxAge  time.Duration
	DryRun                bool
}

type RetentionPlan struct {
	AuditEventsDeleted     int
	RuntimeEventsDeleted   int
	QualityHistoryDeleted  int
	DryRun                 bool
}
```

- [x] **Step 3: 实现 `PlanRetention` 与 `ApplyRetention`**

规则：
- `DryRun=true` 时只统计，不删除。
- `QualityHistoryMaxAge=0` 表示不处理该类数据。
- apply 后写一条 audit event，action 为 `storage.retention.apply`。

Run:

```powershell
go test ./internal/storage -run "TestRetentionPlanDryRunDoesNotDeleteRecentQualityHistory|TestRetentionApplyWritesAudit" -count=1
```

Expected: PASS。

- [x] **Step 4: CLI 命令**

新增：

```powershell
agent data retention --config agent.json --quality-days 30 --dry-run
agent data retention --config agent.json --quality-days 30 --apply
agent data export --config agent.json --out agent-export.json
```

验证：

```powershell
go test ./internal/cli -run TestDataRetentionCommand -count=1
```

Expected: PASS。

---

## Task 6: P11 总验收与文档索引同步

**Files:**
- Modify: `legion/legionAgent/.github/workflows/agent-ci.yml`
- Modify: `docs/agents/legion-agent/index.md`
- Modify: `docs/plans/03-agent/index.md`
- Modify: `docs/plans/03-agent/task-breakdown.md`
- Modify: `docs/plans/03-agent/p11-platform-integration-plan.md`

- [x] **Step 1: CI 增加 P11 门禁**

在 `agent-ci.yml` 增加：

```yaml
- name: OpenAPI compatibility
  run: go test ./internal/compat -run TestOpenAPIGolden -count=1

- name: Platform integration checks
  run: go test ./internal/server ./internal/storage -run "TestSSEEventsFiltersAndSanitizesPayload|TestHTTPRejectsCrossCompanyTaskAccess|TestMetricsPrometheusFormat|TestRetention" -count=1
```

- [x] **Step 2: 文档索引**

更新：
- `docs/agents/legion-agent/index.md` 增加 `openapi.md`、`events.md`、`security-tenancy.md`、`observability.md`、`data-retention.md`。
- `docs/plans/03-agent/index.md` 增加 P11 主线。
- `docs/plans/03-agent/task-breakdown.md` 增加 P11 任务详情。

- [x] **Step 3: 总验证**

Run:

```powershell
go test ./...
go vet ./...
go build -o NUL ./cmd
.\scripts\smoke.ps1
.\scripts\release.ps1 -Version 0.1.0-local -Commit local-test -OutDir .\dist
```

Expected: 全部 PASS。若 release 在 `modernc.org/sqlite` 的 `linux/arm64` 交叉编译出现一次性 Go compiler ICE，先单独复现该目标；若单独目标通过，再完整 release 复跑一次，并在验收记录中注明。

**2026-05-16 验收记录：**

```powershell
go test ./internal/compat -run TestOpenAPIGolden -count=1
go test ./internal/server ./internal/storage -run "TestSSEEventsFiltersAndSanitizesPayload|TestHTTPRejectsCrossCompanyTaskAccess|TestMetricsPrometheusFormat|TestRetention" -count=1
go test ./...
go vet ./...
go build -o NUL ./cmd
.\scripts\smoke.ps1
.\scripts\release.ps1 -Version 0.1.0-local -Commit local-test -OutDir .\dist
```

结果：全部通过，release 产物包含 `windows-amd64`、`linux-amd64`、`linux-arm64`。

## 风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| OpenAPI 过早过细 | 后续演进被锁死 | P11 只锁核心路径、schema 名和敏感字段规则，不锁内部字段全集 |
| SSE 泄露 prompt/tool input | 安全事故 | EventEnvelope 写出前统一 sanitizer，测试覆盖 prompt/secret/input |
| company 边界破坏现有 demo | smoke 失败 | 未提供 `X-Company-ID` 时仅允许 demo/local 明确配置，生产默认拒绝 |
| Prometheus 字段膨胀 | 兼容性负担 | 只导出聚合 counter/gauge，不导出高基数字段 |
| Retention 误删审计证据 | 合规风险 | dry-run 默认，apply 必须显式，操作摘要写 X02 audit |

## 完成定义

P11 完成时，Agent 应满足：

1. HTTP 核心 API 可通过 `/openapi.json` 被外部系统发现，并有 golden 兼容门禁。
2. task/workflow/quality 事件可通过 `/v1/events` 订阅，输出不暴露 prompt、secret、tool input。
3. task/workflow/quality/audit 查询具备 company 作用域保护，跨 company 访问拒绝并审计。
4. `/metrics` 同时支持 JSON 与 Prometheus text format。
5. SQLite 运行事件、质量历史、审计记录具备 dry-run retention 和 apply 归档入口。
6. 文档、CI、任务表同步，`go test ./...`、`go vet ./...`、`go build`、smoke、release 全部通过。
