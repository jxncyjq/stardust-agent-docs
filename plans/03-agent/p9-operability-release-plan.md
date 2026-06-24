---
id: "plans-agent-p9-operability-release-001"
title: "Agent P9 运维化与发布准备计划"
type: "implementation-plan"
category: "plans/agent"
tags: ["plan", "agent", "p9", "operability", "release", "observability", "security"]
version: "0.1.0"
created: "2026-05-13"
updated: "2026-05-13"
status: "draft"
related_docs:
  - path: "./task-breakdown.md"
    relation: "updates"
  - path: "../../design/architecture/agent_components/index.md"
    relation: "derived_from"
  - path: "../../design/architecture/platform-component-registry.md"
    relation: "derived_from"
---

# Agent P9 运维化与发布准备计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 P2-P8 已完成的 Agent 独立服务推进到可部署、可观测、可诊断、可发布、可回滚的上线准备状态。

**Architecture:** P9 不新增核心业务闭环，主要在现有 `internal/server`、`internal/service`、`internal/config`、`internal/storage`、`internal/app` 外围补稳定运维面。安全、观测、发布和兼容性都通过小包和端口注入，不让 AgentRuntime、WorkflowEngine、Memory、Evolution 等核心业务包直接绑定部署细节。

**Tech Stack:** Go 1.26.0、Cobra、net/http、slog、SQLite、PowerShell smoke、GitHub Actions。

---

## P9 范围边界

| 包含 | 不包含 |
|------|--------|
| HTTP API 管理令牌、请求追踪、结构化日志 | 多租户 RBAC 工作台 |
| 指标采集、诊断快照、健康检查增强 | Prometheus 生产部署 Helm chart |
| SQLite schema 版本、备份、恢复演练 | 分布式数据库迁移系统 |
| 版本命令、发布构建矩阵、发布归档 | 自动发布到公网包管理平台 |
| HTTP/API/config/workflow 兼容性测试 | Agent 业务组件重新设计 |
| 运维 runbook 和发布手册 | MaaS/Know 模块实现 |

## 文件结构规划

| 路径 | 动作 | 职责 |
|------|------|------|
| `legion/legionAgent/internal/config/config.go` | Modify | 增加 service admin token、observability、release 配置 |
| `legion/legionAgent/internal/server/http.go` | Modify | 接入认证中间件、request_id、metrics、diagnostics、readiness |
| `legion/legionAgent/internal/server/*_test.go` | Modify/Create | 覆盖认证、健康检查、metrics、兼容性 |
| `legion/legionAgent/internal/observability/metrics.go` | Create | 内存指标采集器，记录任务、HTTP、模型、审批、工作流事件 |
| `legion/legionAgent/internal/observability/logging.go` | Create | slog logger 构造、request/task/component 字段规范 |
| `legion/legionAgent/internal/observability/diagnostics.go` | Create | 运行时诊断快照结构，不暴露密钥和敏感 prompt |
| `legion/legionAgent/internal/storage/migration.go` | Create | SQLite schema 版本管理和幂等迁移 |
| `legion/legionAgent/internal/storage/backup.go` | Create | SQLite 备份、校验和恢复前验证 |
| `legion/legionAgent/internal/version/version.go` | Create | 版本、commit、build time 注入点 |
| `legion/legionAgent/internal/cli/command.go` | Modify | 增加 `version`、`backup`、`restore` 子命令 |
| `legion/legionAgent/.github/workflows/agent-ci.yml` | Modify | 增加 release build matrix 和 artifact smoke |
| `legion/legionAgent/scripts/release.ps1` | Create | 本地 release 构建入口 |
| `docs/agents/legion-agent/operations.md` | Create | 服务部署、健康检查、日志、指标、备份恢复 runbook |
| `docs/agents/legion-agent/release.md` | Create | 版本、构建、产物、回滚说明 |

## 阶段验收门槛

| 门槛 | 命令或检查 |
|------|------------|
| 全量测试 | `go test ./...` |
| 静态检查 | `go vet ./...` |
| 构建验证 | `go build -o NUL ./cmd` |
| smoke | `.\scripts\smoke.ps1` |
| 发布脚本 | `.\scripts\release.ps1 -Version 0.1.0-local -OutDir .\dist` |
| API 认证 | 未带 token 的管理接口返回 401，健康探针可按配置公开 |
| 敏感信息 | diagnostics/logs 不输出 API key、admin token、完整 prompt |

---

## Task 1: HTTP 管理令牌与请求追踪

**Files:**
- Modify: `legion/legionAgent/internal/config/config.go`
- Modify: `legion/legionAgent/internal/server/http.go`
- Test: `legion/legionAgent/internal/server/http_auth_test.go`
- Modify: `docs/agents/legion-agent/configuration.md`

- [x] **Step 1: 写失败测试**

测试点：
- `GET /healthz` 默认可公开访问。
- `POST /v1/tasks` 在配置了 admin token 时，无 `Authorization: Bearer <token>` 返回 401。
- 带正确 token 时可提交任务。
- 响应包含稳定 `X-Request-ID`。

Run:

```powershell
go test ./internal/server -run TestHTTPAdminTokenAndRequestID -count=1
```

Expected: FAIL，原因是当前 server 尚未实现 admin token 和 request id。

- [x] **Step 2: 增加配置字段**

在 `config.ServiceConfig` 增加：

```go
AdminToken           string `json:"admin_token"`
PublicHealthEnabled  bool   `json:"public_health_enabled"`
RequestIDHeader      string `json:"request_id_header"`
```

默认值：

```go
PublicHealthEnabled: true
RequestIDHeader: "X-Request-ID"
```

- [x] **Step 3: 实现中间件**

在 `internal/server/http.go` 增加：

```go
func requireAdminToken(next http.Handler, token string) http.Handler
func withRequestID(next http.Handler, header string) http.Handler
```

规则：
- token 为空时保持开发模式开放，但写 warning log。
- token 非空时，除公开健康探针外所有管理接口都要校验 Bearer token。
- request id 优先使用入站 header，没有则生成本地唯一值。

- [x] **Step 4: 更新配置文档**

在 `docs/agents/legion-agent/configuration.md` 写明：

```json
{
  "service": {
    "admin_token": "change-me",
    "public_health_enabled": true,
    "request_id_header": "X-Request-ID"
  }
}
```

- [x] **Step 5: 验证**

Run:

```powershell
go test ./internal/server -run TestHTTPAdminTokenAndRequestID -count=1
go test ./...
```

Expected: PASS。

---

## Task 2: 结构化日志与组件字段规范

**Files:**
- Create: `legion/legionAgent/internal/observability/logging.go`
- Test: `legion/legionAgent/internal/observability/logging_test.go`
- Modify: `legion/legionAgent/internal/service/service.go`
- Modify: `legion/legionAgent/internal/server/http.go`
- Modify: `legion/legionAgent/internal/app/app.go`

- [x] **Step 1: 写失败测试**

测试点：
- logger 输出 JSON。
- 每条服务日志包含 `level`、`msg`、`component`。
- request 相关日志包含 `request_id`。
- task 相关日志包含 `task_id`。

Run:

```powershell
go test ./internal/observability -run TestLoggerAddsRequiredFields -count=1
```

Expected: FAIL，原因是包不存在。

- [x] **Step 2: 建立 logging 包**

实现：

```go
type LoggerConfig struct {
	Level  string
	Format string
}

func NewLogger(w io.Writer, cfg LoggerConfig) (*slog.Logger, error)
func WithComponent(logger *slog.Logger, component string) *slog.Logger
func WithRequestID(logger *slog.Logger, requestID string) *slog.Logger
func WithTaskID(logger *slog.Logger, taskID string) *slog.Logger
```

规则：
- 默认 JSON。
- 默认 level 为 `info`。
- 不接受未知 level，返回可包装错误。

- [x] **Step 3: 接入服务启动日志**

服务启动、停止、HTTP 请求、task submit、workflow waiting 查询都写结构化日志。日志字段不包含 prompt 全文、API key、admin token。

- [x] **Step 4: 验证**

Run:

```powershell
go test ./internal/observability ./internal/server ./internal/service -count=1
go test ./...
```

Expected: PASS。

---

## Task 3: 内存指标与 `/metrics` 诊断面

**Files:**
- Create: `legion/legionAgent/internal/observability/metrics.go`
- Test: `legion/legionAgent/internal/observability/metrics_test.go`
- Modify: `legion/legionAgent/internal/server/http.go`
- Modify: `legion/legionAgent/internal/app/app.go`
- Modify: `docs/agents/legion-agent/http-api.md`

- [x] **Step 1: 写失败测试**

测试点：
- `MetricsRecorder` 能累加 task submitted/done/failed。
- 能记录 HTTP status 计数。
- 能导出 deterministic JSON 快照。
- `/metrics` 返回 JSON，字段稳定。

Run:

```powershell
go test ./internal/observability -run TestMetricsRecorderSnapshot -count=1
go test ./internal/server -run TestMetricsEndpointReturnsSnapshot -count=1
```

Expected: FAIL。

- [x] **Step 2: 实现指标采集器**

结构：

```go
type MetricsSnapshot struct {
	StartedAt        time.Time        `json:"started_at"`
	Tasks            map[string]int64 `json:"tasks"`
	HTTPStatus       map[string]int64 `json:"http_status"`
	ModelCalls       map[string]int64 `json:"model_calls"`
	Approvals        map[string]int64 `json:"approvals"`
	WorkflowRuns     map[string]int64 `json:"workflow_runs"`
}
```

方法：

```go
func NewMetricsRecorder(now func() time.Time) *MetricsRecorder
func (m *MetricsRecorder) IncTaskStatus(status string)
func (m *MetricsRecorder) IncHTTPStatus(status int)
func (m *MetricsRecorder) Snapshot() MetricsSnapshot
```

- [x] **Step 3: 接入 HTTP server**

新增：

```text
GET /metrics
```

响应使用 `application/json`，受 admin token 保护。

- [x] **Step 4: 更新 HTTP API 文档**

在 `docs/agents/legion-agent/http-api.md` 增加 `/metrics` 响应示例。

- [x] **Step 5: 验证**

Run:

```powershell
go test ./internal/observability ./internal/server -count=1
go test ./...
```

Expected: PASS。

---

## Task 4: Readiness、诊断快照与敏感信息净化

**Files:**
- Create: `legion/legionAgent/internal/observability/diagnostics.go`
- Test: `legion/legionAgent/internal/observability/diagnostics_test.go`
- Modify: `legion/legionAgent/internal/server/http.go`
- Modify: `legion/legionAgent/internal/storage/sqlite.go`
- Modify: `docs/agents/legion-agent/http-api.md`

- [x] **Step 1: 写失败测试**

测试点：
- `/readyz` 在 storage 可用时返回 200。
- storage 不可用时返回 503 和失败原因代码。
- `/debug/diagnostics` 不包含 `api_key`、`admin_token`、完整 prompt。

Run:

```powershell
go test ./internal/server -run "TestReadyz|TestDiagnosticsRedactsSecrets" -count=1
```

Expected: FAIL。

- [x] **Step 2: 增加 storage ping**

在 SQLite repository 增加：

```go
func (r *SQLiteRepository) Ping(ctx context.Context) error
```

readiness 只依赖轻量查询，不触发迁移或写入。

- [x] **Step 3: 实现诊断快照**

快照包含：
- version
- uptime
- config profile，不含 secret
- storage driver/path hash
- scheduler enabled/running
- recent counters

- [x] **Step 4: 接入 HTTP endpoint**

新增：

```text
GET /readyz
GET /debug/diagnostics
```

`/debug/diagnostics` 受 admin token 保护。

- [x] **Step 5: 验证**

Run:

```powershell
go test ./internal/observability ./internal/server ./internal/storage -count=1
go test ./...
```

Expected: PASS。

---

## Task 5: SQLite schema 版本、备份与恢复命令

**Files:**
- Create: `legion/legionAgent/internal/storage/migration.go`
- Create: `legion/legionAgent/internal/storage/backup.go`
- Test: `legion/legionAgent/internal/storage/migration_test.go`
- Test: `legion/legionAgent/internal/storage/backup_test.go`
- Modify: `legion/legionAgent/internal/cli/command.go`
- Test: `legion/legionAgent/internal/cli/command_test.go`
- Create: `docs/agents/legion-agent/storage-ops.md`

- [x] **Step 1: 写失败测试**

测试点：
- 新库初始化后 `schema_version` 为当前版本。
- 重复迁移幂等。
- backup 文件带 checksum。
- restore 前校验 checksum，失败时不覆盖目标库。

Run:

```powershell
go test ./internal/storage -run "TestSchemaMigration|TestSQLiteBackupRestore" -count=1
```

Expected: FAIL。

- [x] **Step 2: 实现迁移表**

创建：

```sql
CREATE TABLE IF NOT EXISTS schema_migrations (
  version INTEGER PRIMARY KEY,
  applied_at TEXT NOT NULL
);
```

当前版本从已有 schema 定义为 `1`。

- [x] **Step 3: 实现备份恢复**

命令：

```powershell
go run ./cmd -- backup --config configs/local.json --out .\backups\agent.db.bak
go run ./cmd -- restore --config configs/local.json --in .\backups\agent.db.bak
```

恢复规则：
- 目标服务必须停止，由 CLI 检测 lock file 或打开独占连接。
- checksum 不匹配直接失败。
- 恢复前创建 `.pre-restore` 备份。

- [x] **Step 4: 写文档**

`storage-ops.md` 写清备份频率、恢复步骤、失败处理、验证命令。

- [x] **Step 5: 验证**

Run:

```powershell
go test ./internal/storage ./internal/cli -count=1
go test ./...
```

Expected: PASS。

---

## Task 6: 版本命令与发布构建脚本

**Files:**
- Create: `legion/legionAgent/internal/version/version.go`
- Test: `legion/legionAgent/internal/version/version_test.go`
- Modify: `legion/legionAgent/internal/cli/command.go`
- Test: `legion/legionAgent/internal/cli/command_test.go`
- Create: `legion/legionAgent/scripts/release.ps1`
- Modify: `legion/legionAgent/.github/workflows/agent-ci.yml`
- Create: `docs/agents/legion-agent/release.md`

- [x] **Step 1: 写失败测试**

测试点：
- `agent version --plain` 输出 version、commit、build_time。
- 默认 dev 值稳定。
- ldflags 注入后输出注入值。

Run:

```powershell
go test ./internal/version ./internal/cli -run TestVersionCommand -count=1
```

Expected: FAIL。

- [x] **Step 2: 实现 version 包**

变量：

```go
var (
	Version   = "dev"
	Commit    = "unknown"
	BuildTime = "unknown"
)
```

方法：

```go
func Info() BuildInfo
```

- [x] **Step 3: 增加 CLI version 命令**

命令：

```powershell
go run ./cmd -- version --plain
```

输出：

```text
version=dev commit=unknown build_time=unknown
```

- [x] **Step 4: 实现 release 脚本**

`scripts/release.ps1` 参数：

```powershell
param(
  [string]$Version,
  [string]$Commit = "local",
  [string]$OutDir = ".\dist"
)
```

构建目标：
- `windows-amd64`
- `linux-amd64`
- `linux-arm64`

- [x] **Step 5: 更新 CI**

CI 增加 release build matrix，但只上传本地 artifact，不自动发布。

- [x] **Step 6: 验证**

Run:

```powershell
go test ./internal/version ./internal/cli -count=1
.\scripts\release.ps1 -Version 0.1.0-local -OutDir .\dist
```

Expected: PASS，`dist` 下生成三个平台产物。

---

## Task 7: API、配置、Workflow DSL 兼容性门禁

**Files:**
- Create: `legion/legionAgent/internal/compat/golden_test.go`
- Create: `legion/legionAgent/internal/compat/testdata/config-minimal.json`
- Create: `legion/legionAgent/internal/compat/testdata/workflow-minimal.json`
- Create: `legion/legionAgent/internal/compat/testdata/http-openapi-lite.json`
- Modify: `legion/legionAgent/.github/workflows/agent-ci.yml`
- Modify: `docs/agents/legion-agent/ci.md`

- [x] **Step 1: 写失败测试**

测试点：
- 最小 config 能被当前 config loader 读取。
- 最小 workflow DSL 能被 WorkflowEngine 解析并运行到 waiting/done。
- HTTP API 响应字段不删除核心字段：`task_id`、`status`、`events`、`workflow_id`。

Run:

```powershell
go test ./internal/compat -count=1
```

Expected: FAIL。

- [x] **Step 2: 建立 compat 包**

测试只依赖公开构造函数和 HTTP 层，不读取内部未导出状态。

- [x] **Step 3: 接入 CI**

CI 增加：

```yaml
- name: Compatibility
  run: go test ./internal/compat -count=1
```

- [x] **Step 4: 更新 CI 文档**

说明 compat 失败代表破坏配置、HTTP 或 DSL 兼容性，需要显式升级版本并写迁移说明。

- [x] **Step 5: 验证**

Run:

```powershell
go test ./internal/compat -count=1
go test ./...
```

Expected: PASS。

---

## Task 8: 运维 Runbook 与 P9 总验收

**Files:**
- Create: `docs/agents/legion-agent/operations.md`
- Modify: `docs/agents/legion-agent/index.md`
- Modify: `docs/plans/03-agent/task-breakdown.md`
- Modify: `docs/plans/03-agent/index.md`
- Modify: `legion/legionAgent/README.md`

- [x] **Step 1: 写 operations 文档**

必须包含：
- 本地启动
- 配置 admin token
- 健康检查
- readiness 检查
- metrics 查看
- diagnostics 获取
- 日志字段说明
- SQLite 备份
- SQLite 恢复
- 发布产物选择
- 回滚步骤

- [x] **Step 2: 更新索引**

更新：
- `docs/agents/legion-agent/index.md`
- `docs/plans/03-agent/index.md`
- `legion/legionAgent/README.md`

- [x] **Step 3: 更新 task-breakdown**

P9 所有 AG-P9-* 在实现完成后标记为 `done`。规划阶段先保持 `todo`。

- [x] **Step 4: 总验证**

Run:

```powershell
go test ./...
go vet ./...
go build -o NUL ./cmd
.\scripts\smoke.ps1
.\scripts\release.ps1 -Version 0.1.0-local -OutDir .\dist
```

Expected: 全部 PASS。

## 风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| 管理接口加 token 后破坏现有 smoke | CI 失败 | health 默认公开，smoke 使用空 token 开发模式；新增认证测试覆盖生产模式 |
| metrics 过早绑定 Prometheus | 引入不必要依赖 | P9 仅输出 JSON 快照，后续再加 Prometheus adapter |
| diagnostics 泄露密钥或 prompt | 安全事故 | 单测固定敏感字段红线，输出前集中 redaction |
| backup/restore 在运行中覆盖 DB | 数据损坏 | restore 要求独占连接，失败时不覆盖目标 |
| release matrix 增加 CI 时间 | 反馈变慢 | matrix 只做 build，不跑重复全量测试 |
| compat 测试过脆 | 阻塞合理演进 | 只锁核心字段和最小契约，不锁完整响应顺序或非关键字段 |

## 完成定义

P9 完成时，Agent 应满足：

1. 服务 API 有最小生产安全边界。
2. 每个请求和任务可以通过 request_id/task_id 追踪。
3. 运维人员能通过 health/readiness/metrics/diagnostics 判断状态。
4. SQLite 数据有 schema version、备份和恢复演练。
5. CI 能验证 test/vet/build/smoke/compat/release build。
6. 发布产物有版本信息和回滚手册。
7. P2-P9 所有计划项在 `task-breakdown.md` 中可追踪。
