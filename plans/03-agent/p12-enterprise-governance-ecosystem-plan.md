---
id: "plans-agent-p12-enterprise-governance-ecosystem-001"
title: "Agent P12 企业治理与外部生态硬化计划"
type: "implementation-plan"
category: "plans/agent"
tags: ["plan", "agent", "p12", "rbac", "skill-registry", "model-profile", "trace", "openapi"]
version: "0.1.0"
created: "2026-05-16"
updated: "2026-05-16"
status: "done"
related_docs:
  - path: "../../design/architecture/agent_components/index.md"
    relation: "derived_from"
  - path: "../../design/architecture/agent_components/agent-component-registry.md"
    relation: "derived_from"
  - path: "../../design/architecture/common_components/index.md"
    relation: "references"
  - path: "../../design/architecture/maas_components/maas-inference-client-spec.md"
    relation: "references"
  - path: "./p11-platform-integration-plan.md"
    relation: "follows"
---

# Agent P12 Enterprise Governance Ecosystem Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 P11 的平台集成就绪能力继续推进到企业治理和外部生态可接入状态，补齐 RBAC 查询边界、远端 Skill registry、模型 profile 路由、trace 出口和错误契约门禁。

**Architecture:** P12 继续保持 A00-A70 组件边界不倒挂：治理能力放在 `internal/security`、`internal/server` 和 `internal/observability`，外部生态通过稳定端口接入。Skill 远端同步复用 A31/A32 与 X03/X04/X02，模型 profile 只扩展 C70 端口装配，不让 runtime 绑定供应商细节。所有新增 HTTP 面都进入 OpenAPI golden，避免 P12 之后客户端契约漂移。

**Tech Stack:** Go 1.26.0、net/http、SQLite、JSON OpenAPI 3.1、PowerShell、GitHub Actions、server-side RBAC、structured trace JSON。

---

## P12 定位

P11 已完成 OpenAPI、SSE、tenant/company 边界、Prometheus、数据保留与总验收。P12 的定位是 **enterprise governance and ecosystem hardening**：

- 让 quality/audit 查询真正具备 company + role 边界，而不是只保护 task/workflow。
- 让 A30-A32 能从远端 registry 同步技能，但仍走 hash、扫描、quarantine、审计。
- 让 C70 MaaS 推理端口支持多模型 profile 和显式路由，便于 Agent/Know/Aegis/SemanticExtractor 共用。
- 让观测从 metrics 推进到 trace 摘要出口，但不引入外部 OTel 后端。
- 让 OpenAPI 不只锁路径，还锁错误码矩阵和客户端兼容样例。

P12 不做这些事：

- 不接入真实公网技能市场账号体系，只实现 HTTP/文件 registry manifest 同步协议和安全门禁。
- 不实现完整 IAM 产品，只提供 Agent 服务层必须使用的 role/action/resource 判定。
- 不引入生产级分布式 tracing 后端，只导出结构化 trace snapshot 和兼容测试。
- 不重写 MaaS 推理协议，只在 C70 调用前增加 profile 选择和配置装配。

## 设计依据

| 设计来源 | P12 关注点 |
|----------|------------|
| `agent_components/index.md` | A20-A23 工具安全、A30-A32 技能生态、A60-A64 质量治理、A70 工作流服务面 |
| `common_components/index.md` | X02 审计、X03 安全抓取、X04 路径约束、X05 输出净化 |
| `maas-inference-client-spec.md` | C70 作为稳定推理端口，避免 runtime 绑定 provider 细节 |
| P11 platform integration | 已有 API/SSE/metrics/tenant 基础，P12 在其上增加企业治理面 |

## 文件结构规划

| 路径 | 动作 | 职责 |
|------|------|------|
| `legion/legionAgent/internal/security/rbac.go` | Create | 定义 role、action、resource 与 RBAC 判定器 |
| `legion/legionAgent/internal/security/rbac_test.go` | Create | 验证 admin/operator/viewer 对 audit/quality/task/workflow 的访问矩阵 |
| `legion/legionAgent/internal/server/governance.go` | Create | `/v1/audit-events`、`/v1/quality/evals` 查询接口和 company 过滤 |
| `legion/legionAgent/internal/server/governance_test.go` | Create | 验证 RBAC、company 过滤、敏感字段不输出 |
| `legion/legionAgent/internal/skill/registry_sync.go` | Create | 远端 registry index 拉取、manifest 校验、安装调度 |
| `legion/legionAgent/internal/skill/registry_sync_test.go` | Create | 使用 `httptest.Server` 验证 registry sync、hash mismatch、critical quarantine |
| `legion/legionAgent/internal/config/config.go` | Modify | 增加 `maas.profiles`、`maas.default_profile`、`skills.registry_url` 配置 |
| `legion/legionAgent/internal/adapter/maas_profile.go` | Create | 基于 profile 装配 C70 HTTP MaaS client |
| `legion/legionAgent/internal/adapter/maas_profile_test.go` | Create | 验证 profile 选择、fallback、API key 不泄露 |
| `legion/legionAgent/internal/observability/trace.go` | Create | 轻量 trace recorder、span snapshot、敏感字段净化 |
| `legion/legionAgent/internal/observability/trace_test.go` | Create | 验证 span 顺序、duration、prompt/API key 不泄露 |
| `legion/legionAgent/internal/server/trace_test.go` | Create | 验证 `/debug/traces` token 保护与 JSON 输出 |
| `legion/legionAgent/internal/server/openapi.go` | Modify | 增加 governance、trace、skill registry、错误响应 schema |
| `legion/legionAgent/internal/compat/openapi_error_golden_test.go` | Create | 锁定错误码矩阵和关键客户端请求/响应样例 |
| `docs/agents/legion-agent/governance-rbac.md` | Create | RBAC、company 过滤、审计查询操作说明 |
| `docs/agents/legion-agent/skill-registry.md` | Create | 远端技能 registry manifest、hash、扫描、quarantine 说明 |
| `docs/agents/legion-agent/model-profiles.md` | Create | C70 多模型 profile 配置与路由说明 |
| `docs/agents/legion-agent/traces.md` | Create | trace snapshot 字段、脱敏与运维说明 |

## P12 任务清单

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P12-001 | P0 | Security/X02/A60-A64 | RBAC 与 audit/quality 查询边界 | `done` | `internal/security`, `internal/server`, `internal/storage` | audit/quality 查询按 role 控制，拒绝写 X02；company schema 扩展前保持 task/workflow company 边界 |
| AG-P12-002 | P0 | A30-A32/X03/X04/X02 | 远端 Skill registry 同步 | `done` | `internal/skill`, `internal/config`, `internal/cli`, `docs/agents/legion-agent` | registry sync 走 hash、扫描、quarantine、审计 |
| AG-P12-003 | P1 | C70/A01/A60 | MaaS 多模型 profile 与路由 | `done` | `internal/config`, `internal/adapter`, `internal/cli` | CLI/config 可选择 profile，runtime 仍只依赖 C70 |
| AG-P12-004 | P1 | Observability/X00/X05 | Trace recorder 与 `/debug/traces` | `done` | `internal/observability`, `internal/server` | trace snapshot 可导出，敏感字段被净化 |
| AG-P12-005 | P1 | API/Compat/X05 | OpenAPI 错误矩阵与客户端兼容样例 | `done` | `internal/server`, `internal/compat`, `docs/agents/legion-agent` | 错误响应 schema、HTTP status、示例请求响应有 golden 门禁 |
| AG-P12-006 | P1 | CI/Docs | P12 总验收与文档索引同步 | `done` | `.github/workflows`, `docs/agents/legion-agent`, `docs/plans/03-agent` | P12 专项、test/vet/build/smoke/release 全部通过 |

---

## Task 1: RBAC 与 audit/quality 查询边界

**Files:**
- Create: `legion/legionAgent/internal/security/rbac.go`
- Create: `legion/legionAgent/internal/security/rbac_test.go`
- Create: `legion/legionAgent/internal/server/governance.go`
- Create: `legion/legionAgent/internal/server/governance_test.go`
- Modify: `legion/legionAgent/internal/server/http.go`
- Modify: `legion/legionAgent/internal/server/openapi.go`
- Create: `docs/agents/legion-agent/governance-rbac.md`

- [x] **Step 1: 写 RBAC 失败测试**

在 `internal/security/rbac_test.go` 添加：

```go
package security

import "testing"

func TestRBACAllowsViewerReadQualityButRejectsAudit(t *testing.T) {
	t.Parallel()
	policy := NewRBACPolicy()
	viewer := Principal{CompanyID: "company-1", Role: "viewer"}
	if !policy.Allows(viewer, ActionReadQuality, ResourceQuality) {
		t.Fatalf("Allows(viewer, read_quality, quality) = false, want true")
	}
	if policy.Allows(viewer, ActionReadAudit, ResourceAudit) {
		t.Fatalf("Allows(viewer, read_audit, audit) = true, want false")
	}
}
```

- [x] **Step 2: 运行测试确认失败**

```powershell
go test ./internal/security -run TestRBACAllowsViewerReadQualityButRejectsAudit -count=1
```

Expected: FAIL，`NewRBACPolicy`、`ActionReadQuality`、`ResourceQuality` 尚未定义。

- [x] **Step 3: 实现最小 RBAC policy**

在 `internal/security/rbac.go` 添加：

```go
package security

type Action string

const (
	ActionReadAudit   Action = "read_audit"
	ActionReadQuality Action = "read_quality"
	ActionReadTask    Action = "read_task"
	ActionReadWorkflow Action = "read_workflow"
)

type Resource string

const (
	ResourceAudit    Resource = "audit"
	ResourceQuality  Resource = "quality"
	ResourceTask     Resource = "task"
	ResourceWorkflow Resource = "workflow"
)

type RBACPolicy struct{}

func NewRBACPolicy() RBACPolicy {
	return RBACPolicy{}
}

func (RBACPolicy) Allows(principal Principal, action Action, resource Resource) bool {
	switch principal.Role {
	case "admin":
		return true
	case "operator":
		return resource != ResourceAudit || action == ActionReadAudit
	case "viewer":
		return action == ActionReadQuality || action == ActionReadTask || action == ActionReadWorkflow
	default:
		return false
	}
}
```

- [x] **Step 4: 运行 RBAC 测试**

```powershell
go test ./internal/security -run TestRBACAllowsViewerReadQualityButRejectsAudit -count=1
```

Expected: PASS。

- [x] **Step 5: 写 HTTP governance 失败测试**

在 `internal/server/governance_test.go` 添加：

```go
package server

import (
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"

	"github.com/stardust/legion-agent/internal/adapter"
	"github.com/stardust/legion-agent/internal/domain"
)

func TestHTTPAuditEventsRequireAdminRole(t *testing.T) {
	t.Parallel()
	audit := adapter.NewMemoryAuditLog()
	if err := audit.Append(t.Context(), domain.AuditEvent{
		ID: "audit-1", RequestID: "req-1", SubjectType: "task", SubjectID: "task-1",
		Action: "task.created", Hash: "hash", CreatedAt: time.Now(),
	}); err != nil {
		t.Fatalf("Append(audit-1) error = %v, want nil", err)
	}
	srv := NewHTTPServer(Config{AdminToken: "token", Audit: audit})

	req := httptest.NewRequest(http.MethodGet, "/v1/audit-events", nil)
	req.Header.Set("Authorization", "Bearer token")
	req.Header.Set("X-Company-ID", "company-1")
	req.Header.Set("X-Role", "viewer")
	rec := httptest.NewRecorder()
	srv.ServeHTTP(rec, req)
	if rec.Code != http.StatusForbidden {
		t.Fatalf("GET /v1/audit-events viewer status = %d, want %d body=%s", rec.Code, http.StatusForbidden, rec.Body.String())
	}

	req = httptest.NewRequest(http.MethodGet, "/v1/audit-events", nil)
	req.Header.Set("Authorization", "Bearer token")
	req.Header.Set("X-Company-ID", "company-1")
	req.Header.Set("X-Role", "admin")
	rec = httptest.NewRecorder()
	srv.ServeHTTP(rec, req)
	if rec.Code != http.StatusOK {
		t.Fatalf("GET /v1/audit-events admin status = %d, want %d body=%s", rec.Code, http.StatusOK, rec.Body.String())
	}
	if !strings.Contains(rec.Body.String(), "task.created") {
		t.Fatalf("GET /v1/audit-events body = %s, want audit action", rec.Body.String())
	}
}
```

- [x] **Step 6: 运行 HTTP governance 测试确认失败**

```powershell
go test ./internal/server -run TestHTTPAuditEventsRequireAdminRole -count=1
```

Expected: FAIL，`/v1/audit-events` 尚未路由。

- [x] **Step 7: 实现 `/v1/audit-events` 与 role 判定**

在 `internal/server/governance.go` 添加 `handleAuditEvents`，从请求头读取 `X-Role`，复用 `security.NewRBACPolicy()` 判定 `read_audit`，允许后调用 `ListAuditEvents` 或兼容内存 audit log 的 `Events()`。

- [x] **Step 8: 运行 governance 测试**

```powershell
go test ./internal/security ./internal/server -run "TestRBACAllowsViewerReadQualityButRejectsAudit|TestHTTPAuditEventsRequireAdminRole" -count=1
```

Expected: PASS。

---

## Task 2: 远端 Skill registry 同步

**Files:**
- Create: `legion/legionAgent/internal/skill/registry_sync.go`
- Create: `legion/legionAgent/internal/skill/registry_sync_test.go`
- Modify: `legion/legionAgent/internal/config/config.go`
- Modify: `legion/legionAgent/internal/cli/command.go`
- Create: `docs/agents/legion-agent/skill-registry.md`

- [x] **Step 1: 写 registry sync 失败测试**

在 `internal/skill/registry_sync_test.go` 添加：

```go
package skill

import (
	"net/http"
	"net/http/httptest"
	"path/filepath"
	"testing"
)

func TestRegistrySyncInstallsRemoteManifest(t *testing.T) {
	t.Parallel()
	content := skillDoc("go-testing", "Go Testing", "1.0.0", "safe", "active", "go,test")
	sha := sha256Hex(content)
	var baseURL string
	server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		switch r.URL.Path {
		case "/index.json":
			_, _ = w.Write([]byte(`{"skills":[{"manifest_url":"` + baseURL + `/go-testing.json"}]}`))
		case "/go-testing.json":
			_, _ = w.Write([]byte(`{"id":"go-testing","name":"Go Testing","version":"1.0.0","content_path":"` + baseURL + `/go-testing/SKILL.md","sha256":"` + sha + `"}`))
		case "/go-testing/SKILL.md":
			_, _ = w.Write([]byte(content))
		default:
			http.NotFound(w, r)
		}
	}))
	baseURL = server.URL
	t.Cleanup(server.Close)

	repo := NewMemoryRepository()
	syncer := NewRegistrySyncer(RegistrySyncConfig{
		IndexURL:    server.URL + "/index.json",
		InstallRoot: filepath.Join(t.TempDir(), "skills"),
		Repository: repo,
		Scanner:    NewSecurityScanner(),
	})
	report, err := syncer.Sync(t.Context())
	if err != nil {
		t.Fatalf("Sync() error = %v, want nil", err)
	}
	if report.Installed != 1 {
		t.Fatalf("Sync().Installed = %d, want 1", report.Installed)
	}
}
```

- [x] **Step 2: 运行测试确认失败**

```powershell
go test ./internal/skill -run TestRegistrySyncInstallsRemoteManifest -count=1
```

Expected: FAIL，`RegistrySyncConfig` 和 `NewRegistrySyncer` 尚未定义。

- [x] **Step 3: 实现 RegistrySyncer**

在 `internal/skill/registry_sync.go` 添加：

```go
package skill

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"path/filepath"
)

type RegistrySyncConfig struct {
	IndexURL    string
	InstallRoot string
	Repository  Repository
	Scanner     SkillSecurityScanner
}

type RegistrySyncReport struct {
	Installed   int `json:"installed"`
	Quarantined int `json:"quarantined"`
	Failed      int `json:"failed"`
}

type registryIndex struct {
	Skills []registryIndexItem `json:"skills"`
}

type registryIndexItem struct {
	ManifestURL string `json:"manifest_url"`
}

type RegistrySyncer struct {
	cfg RegistrySyncConfig
}

func NewRegistrySyncer(cfg RegistrySyncConfig) *RegistrySyncer {
	return &RegistrySyncer{cfg: cfg}
}

func (s *RegistrySyncer) Sync(ctx context.Context) (RegistrySyncReport, error) {
	index, err := s.fetchIndex(ctx)
	if err != nil {
		return RegistrySyncReport{}, err
	}
	report := RegistrySyncReport{}
	for _, item := range index.Skills {
		manifestPath, cleanup, err := s.fetchManifest(ctx, item.ManifestURL)
		if err != nil {
			report.Failed++
			continue
		}
		installer := NewInstaller(InstallerConfig{
			InstallRoot: s.cfg.InstallRoot,
			Scanner:     s.cfg.Scanner,
			Repository:  s.cfg.Repository,
		})
		_, err = installer.InstallFromManifest(ctx, manifestPath)
		cleanup()
		if err == nil {
			report.Installed++
			continue
		}
		if errors.Is(err, ErrSkillScanBlocked) {
			report.Quarantined++
			continue
		}
		report.Failed++
	}
	return report, nil
}

func (s *RegistrySyncer) fetchIndex(ctx context.Context) (registryIndex, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, s.cfg.IndexURL, nil)
	if err != nil {
		return registryIndex{}, fmt.Errorf("create registry index request: %w", err)
	}
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return registryIndex{}, fmt.Errorf("fetch registry index: %w", err)
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return registryIndex{}, fmt.Errorf("fetch registry index status %d", resp.StatusCode)
	}
	var index registryIndex
	if err := json.NewDecoder(resp.Body).Decode(&index); err != nil {
		return registryIndex{}, fmt.Errorf("decode registry index: %w", err)
	}
	return index, nil
}

func (s *RegistrySyncer) fetchManifest(ctx context.Context, manifestURL string) (string, func(), error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, manifestURL, nil)
	if err != nil {
		return "", func() {}, fmt.Errorf("create registry manifest request: %w", err)
	}
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return "", func() {}, fmt.Errorf("fetch registry manifest: %w", err)
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return "", func() {}, fmt.Errorf("fetch registry manifest status %d", resp.StatusCode)
	}
	var manifest Manifest
	if err := json.NewDecoder(resp.Body).Decode(&manifest); err != nil {
		return "", func() {}, fmt.Errorf("decode registry manifest: %w", err)
	}
	dir, err := os.MkdirTemp("", "legion-skill-manifest-*")
	if err != nil {
		return "", func() {}, fmt.Errorf("create registry manifest temp dir: %w", err)
	}
	cleanup := func() { _ = os.RemoveAll(dir) }
	content, err := s.fetchBytes(ctx, manifest.ContentPath)
	if err != nil {
		cleanup()
		return "", func() {}, err
	}
	contentPath := filepath.Join(dir, "SKILL.md")
	if err := os.WriteFile(contentPath, content, 0o600); err != nil {
		cleanup()
		return "", func() {}, fmt.Errorf("write registry skill content: %w", err)
	}
	manifest.ContentPath = contentPath
	body, err := json.Marshal(manifest)
	if err != nil {
		cleanup()
		return "", func() {}, fmt.Errorf("marshal local registry manifest: %w", err)
	}
	path := filepath.Join(dir, "manifest.json")
	if err := os.WriteFile(path, body, 0o600); err != nil {
		cleanup()
		return "", func() {}, fmt.Errorf("write registry manifest: %w", err)
	}
	return path, cleanup, nil
}

func (s *RegistrySyncer) fetchBytes(ctx context.Context, url string) ([]byte, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return nil, fmt.Errorf("create registry content request: %w", err)
	}
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, fmt.Errorf("fetch registry content: %w", err)
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("fetch registry content status %d", resp.StatusCode)
	}
	body, err := io.ReadAll(resp.Body)
	if err != nil {
		return nil, fmt.Errorf("read registry content: %w", err)
	}
	return body, nil
}
```

- [x] **Step 4: 增加 CLI 命令**

在 `internal/cli/command.go` 的 `data` 同级增加 `skill sync`：

```powershell
agent skill sync --config agent.json --registry-url http://127.0.0.1:8080/index.json
```

输出：

```text
skill_sync installed=1 quarantined=0 failed=0
```

- [x] **Step 5: 运行 skill 测试**

```powershell
go test ./internal/skill ./internal/cli -run "TestRegistrySyncInstallsRemoteManifest|TestSkillSyncCommand" -count=1
```

Expected: PASS。

---

## Task 3: MaaS 多模型 profile 与路由

**Files:**
- Modify: `legion/legionAgent/internal/config/config.go`
- Modify: `legion/legionAgent/internal/config/config_test.go`
- Create: `legion/legionAgent/internal/adapter/maas_profile.go`
- Create: `legion/legionAgent/internal/adapter/maas_profile_test.go`
- Modify: `legion/legionAgent/internal/cli/command.go`
- Create: `docs/agents/legion-agent/model-profiles.md`

- [x] **Step 1: 写 config 失败测试**

在 `internal/config/config_test.go` 添加：

```go
func TestLoadMaasProfiles(t *testing.T) {
	t.Parallel()
	path := filepath.Join(t.TempDir(), "agent.json")
	body := `{"maas":{"default_profile":"fast","profiles":{"fast":{"base_url":"https://fast.example.test","api_key":"fast-key"},"review":{"base_url":"https://review.example.test","api_key":"review-key"}}}}`
	if err := os.WriteFile(path, []byte(body), 0o600); err != nil {
		t.Fatalf("WriteFile(%q) error = %v, want nil", path, err)
	}
	cfg, err := Load(context.Background(), Options{Path: path})
	if err != nil {
		t.Fatalf("Load(%q) error = %v, want nil", path, err)
	}
	if cfg.Maas.DefaultProfile != "fast" {
		t.Fatalf("Load(%q).Maas.DefaultProfile = %q, want fast", path, cfg.Maas.DefaultProfile)
	}
	if cfg.Maas.Profiles["review"].BaseURL != "https://review.example.test" {
		t.Fatalf("Load(%q).Maas.Profiles[review].BaseURL = %q, want review URL", path, cfg.Maas.Profiles["review"].BaseURL)
	}
}
```

- [x] **Step 2: 运行测试确认失败**

```powershell
go test ./internal/config -run TestLoadMaasProfiles -count=1
```

Expected: FAIL，`DefaultProfile` 和 `Profiles` 尚未定义。

- [x] **Step 3: 扩展配置类型**

在 `internal/config/config.go` 扩展：

```go
type MaasConfig struct {
	BaseURL        string                    `json:"base_url"`
	APIKey         string                    `json:"api_key"`
	DefaultProfile string                    `json:"default_profile"`
	Profiles       map[string]MaasProfile   `json:"profiles"`
}

type MaasProfile struct {
	BaseURL string `json:"base_url"`
	APIKey  string `json:"api_key"`
}
```

默认配置中设置 `Profiles: map[string]MaasProfile{}`，环境变量保持兼容，只覆盖旧字段。

- [x] **Step 4: 写 profile factory 测试**

在 `internal/adapter/maas_profile_test.go` 添加：

```go
package adapter

import (
	"testing"

	"github.com/stardust/legion-agent/internal/config"
)

func TestNewMaasClientFromProfileUsesNamedProfile(t *testing.T) {
	t.Parallel()
	client, err := NewMaasClientFromProfile(config.MaasConfig{
		DefaultProfile: "fast",
		Profiles: map[string]config.MaasProfile{
			"review": {BaseURL: "https://review.example.test", APIKey: "review-key"},
		},
	}, "review")
	if err != nil {
		t.Fatalf("NewMaasClientFromProfile(review) error = %v, want nil", err)
	}
	httpClient, ok := client.(*HTTPMaasClient)
	if !ok {
		t.Fatalf("NewMaasClientFromProfile(review) = %T, want *HTTPMaasClient", client)
	}
	if httpClient.baseURL != "https://review.example.test" {
		t.Fatalf("HTTPMaasClient.baseURL = %q, want review URL", httpClient.baseURL)
	}
}
```

- [x] **Step 5: 实现 profile factory**

在 `internal/adapter/maas_profile.go` 添加：

```go
package adapter

import (
	"fmt"

	"github.com/stardust/legion-agent/internal/config"
	"github.com/stardust/legion-agent/internal/port"
)

func NewMaasClientFromProfile(cfg config.MaasConfig, name string) (port.MaasInferenceClient, error) {
	if name == "" {
		name = cfg.DefaultProfile
	}
	if name != "" {
		profile, ok := cfg.Profiles[name]
		if !ok {
			return nil, fmt.Errorf("maas profile %q not found", name)
		}
		return NewHTTPMaasClient(HTTPMaasConfig{BaseURL: profile.BaseURL, APIKey: profile.APIKey}), nil
	}
	if cfg.BaseURL == "" {
		return nil, nil
	}
	return NewHTTPMaasClient(HTTPMaasConfig{BaseURL: cfg.BaseURL, APIKey: cfg.APIKey}), nil
}
```

- [x] **Step 6: CLI 接入 `--maas-profile`**

在 `agent run` 增加：

```powershell
agent run --config agent.json --maas-profile review --prompt "Review this output" --plain
```

当 `--maas-url` 显式传入时仍优先使用 URL/API key，保证旧脚本兼容。

- [x] **Step 7: 运行配置与 adapter 测试**

```powershell
go test ./internal/config ./internal/adapter ./internal/cli -run "TestLoadMaasProfiles|TestNewMaasClientFromProfileUsesNamedProfile|TestRunCommandUsesMaasProfile" -count=1
```

Expected: PASS。

---

## Task 4: Trace recorder 与 `/debug/traces`

**Files:**
- Create: `legion/legionAgent/internal/observability/trace.go`
- Create: `legion/legionAgent/internal/observability/trace_test.go`
- Create: `legion/legionAgent/internal/server/trace_test.go`
- Modify: `legion/legionAgent/internal/server/http.go`
- Modify: `legion/legionAgent/internal/server/openapi.go`
- Create: `docs/agents/legion-agent/traces.md`

- [x] **Step 1: 写 trace recorder 失败测试**

在 `internal/observability/trace_test.go` 添加：

```go
package observability

import (
	"strings"
	"testing"
	"time"
)

func TestTraceRecorderSanitizesSpanAttributes(t *testing.T) {
	t.Parallel()
	recorder := NewTraceRecorder(TraceConfig{MaxSpans: 10})
	recorder.Record(Span{
		TraceID: "trace-1", SpanID: "span-1", Name: "model.generate",
		StartedAt: time.Now(), EndedAt: time.Now().Add(time.Millisecond),
		Attributes: map[string]string{"prompt": "secret prompt", "component": "runtime"},
	})
	snapshot := recorder.Snapshot()
	if len(snapshot.Spans) != 1 {
		t.Fatalf("Snapshot().Spans len = %d, want 1", len(snapshot.Spans))
	}
	body := snapshot.Spans[0].Attributes["prompt"]
	if strings.Contains(body, "secret prompt") {
		t.Fatalf("Snapshot().Spans[0].Attributes[prompt] = %q, want redacted", body)
	}
}
```

- [x] **Step 2: 运行测试确认失败**

```powershell
go test ./internal/observability -run TestTraceRecorderSanitizesSpanAttributes -count=1
```

Expected: FAIL，trace 类型尚未定义。

- [x] **Step 3: 实现 TraceRecorder**

在 `internal/observability/trace.go` 添加：

```go
package observability

import (
	"strings"
	"sync"
	"time"
)

type TraceConfig struct {
	MaxSpans int
}

type Span struct {
	TraceID    string            `json:"trace_id"`
	SpanID     string            `json:"span_id"`
	Name       string            `json:"name"`
	StartedAt  time.Time         `json:"started_at"`
	EndedAt    time.Time         `json:"ended_at"`
	Attributes map[string]string `json:"attributes"`
}

type TraceSnapshot struct {
	Spans []Span `json:"spans"`
}

type TraceRecorder struct {
	mu       sync.Mutex
	maxSpans int
	spans    []Span
}

func NewTraceRecorder(cfg TraceConfig) *TraceRecorder {
	if cfg.MaxSpans <= 0 {
		cfg.MaxSpans = 100
	}
	return &TraceRecorder{maxSpans: cfg.MaxSpans}
}

func (r *TraceRecorder) Record(span Span) {
	r.mu.Lock()
	defer r.mu.Unlock()
	span.Attributes = sanitizeTraceAttributes(span.Attributes)
	r.spans = append(r.spans, span)
	if len(r.spans) > r.maxSpans {
		r.spans = r.spans[len(r.spans)-r.maxSpans:]
	}
}

func (r *TraceRecorder) Snapshot() TraceSnapshot {
	r.mu.Lock()
	defer r.mu.Unlock()
	spans := make([]Span, len(r.spans))
	copy(spans, r.spans)
	return TraceSnapshot{Spans: spans}
}

func sanitizeTraceAttributes(attrs map[string]string) map[string]string {
	out := make(map[string]string, len(attrs))
	for key, value := range attrs {
		lower := strings.ToLower(key)
		if strings.Contains(lower, "prompt") || strings.Contains(lower, "secret") || strings.Contains(lower, "api_key") || strings.Contains(lower, "token") {
			out[key] = "[redacted]"
			continue
		}
		out[key] = value
	}
	return out
}
```

- [x] **Step 4: 写 HTTP trace 失败测试**

在 `internal/server/trace_test.go` 添加：

```go
package server

import (
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"

	"github.com/stardust/legion-agent/internal/observability"
)

func TestHTTPTracesRequireAdminTokenAndRedactSecrets(t *testing.T) {
	t.Parallel()
	traces := observability.NewTraceRecorder(observability.TraceConfig{MaxSpans: 10})
	traces.Record(observability.Span{
		TraceID: "trace-1", SpanID: "span-1", Name: "model.generate",
		StartedAt: time.Now(), EndedAt: time.Now(),
		Attributes: map[string]string{"api_key": "secret-key"},
	})
	srv := NewHTTPServer(Config{AdminToken: "token", Traces: traces})

	unauthorized := httptest.NewRecorder()
	srv.ServeHTTP(unauthorized, httptest.NewRequest(http.MethodGet, "/debug/traces", nil))
	if unauthorized.Code != http.StatusUnauthorized {
		t.Fatalf("GET /debug/traces without token status = %d, want %d", unauthorized.Code, http.StatusUnauthorized)
	}

	req := httptest.NewRequest(http.MethodGet, "/debug/traces", nil)
	req.Header.Set("Authorization", "Bearer token")
	rec := httptest.NewRecorder()
	srv.ServeHTTP(rec, req)
	if rec.Code != http.StatusOK {
		t.Fatalf("GET /debug/traces status = %d, want %d body=%s", rec.Code, http.StatusOK, rec.Body.String())
	}
	if strings.Contains(rec.Body.String(), "secret-key") {
		t.Fatalf("GET /debug/traces body leaked secret: %s", rec.Body.String())
	}
}
```

- [x] **Step 5: 实现 `/debug/traces`**

在 `internal/server/http.go` 的 `Config` 增加 `Traces *observability.TraceRecorder`，路由增加 `GET /debug/traces`，handler 返回 `TraceSnapshot`。

- [x] **Step 6: 运行 trace 测试**

```powershell
go test ./internal/observability ./internal/server -run "TestTraceRecorderSanitizesSpanAttributes|TestHTTPTracesRequireAdminTokenAndRedactSecrets" -count=1
```

Expected: PASS。

---

## Task 5: OpenAPI 错误矩阵与客户端兼容样例

**Files:**
- Modify: `legion/legionAgent/internal/server/openapi.go`
- Modify: `legion/legionAgent/internal/server/openapi_test.go`
- Create: `legion/legionAgent/internal/compat/openapi_error_golden_test.go`
- Modify: `legion/legionAgent/internal/compat/testdata/openapi-agent.json`
- Create: `docs/agents/legion-agent/api-errors.md`

- [x] **Step 1: 写 OpenAPI 错误矩阵失败测试**

在 `internal/server/openapi_test.go` 添加：

```go
func TestOpenAPISpecIncludesErrorResponses(t *testing.T) {
	t.Parallel()
	spec := AgentOpenAPISpec()
	op := spec.Paths["/v1/tasks"].Post
	for _, status := range []string{"400", "401", "403", "500"} {
		if _, ok := op.Responses[status]; !ok {
			t.Fatalf("POST /v1/tasks response %s missing", status)
		}
	}
	if _, ok := spec.Components.Schemas["ErrorResponse"]; !ok {
		t.Fatalf("components.schemas.ErrorResponse missing")
	}
}
```

- [x] **Step 2: 运行测试确认失败**

```powershell
go test ./internal/server -run TestOpenAPISpecIncludesErrorResponses -count=1
```

Expected: FAIL，错误响应 schema 尚未完整定义。

- [x] **Step 3: 扩展 OpenAPI operation response**

在 `internal/server/openapi.go` 中为受保护接口统一添加：

```json
{
  "400": {"description": "Bad request", "content": {"application/json": {"schema": {"$ref": "#/components/schemas/ErrorResponse"}}}},
  "401": {"description": "Unauthorized", "content": {"application/json": {"schema": {"$ref": "#/components/schemas/ErrorResponse"}}}},
  "403": {"description": "Forbidden", "content": {"application/json": {"schema": {"$ref": "#/components/schemas/ErrorResponse"}}}},
  "500": {"description": "Internal server error", "content": {"application/json": {"schema": {"$ref": "#/components/schemas/ErrorResponse"}}}}
}
```

`ErrorResponse` schema 固定为：

```json
{
  "type": "object",
  "required": ["error"],
  "properties": {
    "error": {"type": "string"}
  }
}
```

- [x] **Step 4: 写兼容性 golden 测试**

在 `internal/compat/openapi_error_golden_test.go` 添加：

```go
package compat

import (
	"encoding/json"
	"os"
	"testing"

	"github.com/stardust/legion-agent/internal/server"
)

func TestOpenAPIErrorContractGolden(t *testing.T) {
	t.Parallel()
	got := server.AgentOpenAPISpec()
	body, err := json.MarshalIndent(got.Components.Schemas["ErrorResponse"], "", "  ")
	if err != nil {
		t.Fatalf("MarshalIndent(ErrorResponse) error = %v, want nil", err)
	}
	want, err := os.ReadFile("testdata/openapi-error-response.json")
	if err != nil {
		t.Fatalf("ReadFile(openapi-error-response.json) error = %v, want nil", err)
	}
	if string(body)+"\n" != string(want) {
		t.Fatalf("ErrorResponse golden mismatch\nwant:\n%s\ngot:\n%s\n", want, body)
	}
}
```

创建 `internal/compat/testdata/openapi-error-response.json`：

```json
{
  "properties": {
    "error": {
      "type": "string"
    }
  },
  "required": [
    "error"
  ],
  "type": "object"
}
```

- [x] **Step 5: 运行 OpenAPI 兼容测试**

```powershell
go test ./internal/server ./internal/compat -run "TestOpenAPISpecIncludesErrorResponses|TestOpenAPIErrorContractGolden|TestOpenAPIGolden" -count=1
```

Expected: PASS。

---

## Task 6: P12 总验收与文档索引同步

**Files:**
- Modify: `legion/legionAgent/.github/workflows/agent-ci.yml`
- Modify: `docs/agents/legion-agent/index.md`
- Modify: `docs/plans/03-agent/index.md`
- Modify: `docs/plans/03-agent/task-breakdown.md`
- Modify: `docs/plans/03-agent/p12-enterprise-governance-ecosystem-plan.md`

- [x] **Step 1: CI 增加 P12 门禁**

在 `agent-ci.yml` 增加：

```yaml
- name: Enterprise governance checks
  run: go test ./internal/security ./internal/server -run "TestRBAC|TestHTTPAuditEvents|TestHTTPTraces" -count=1

- name: Ecosystem checks
  run: go test ./internal/skill ./internal/adapter ./internal/config -run "TestRegistrySync|TestNewMaasClientFromProfile|TestLoadMaasProfiles" -count=1

- name: OpenAPI error compatibility
  run: go test ./internal/compat -run "TestOpenAPIErrorContractGolden|TestOpenAPIGolden" -count=1
```

- [x] **Step 2: 文档索引**

更新 `docs/agents/legion-agent/index.md`，增加：

```markdown
| [governance-rbac.md](./governance-rbac.md) | P12 RBAC、company 过滤、audit/quality 查询边界说明 |
| [skill-registry.md](./skill-registry.md) | P12 远端 Skill registry 同步、hash 校验、扫描与 quarantine 说明 |
| [model-profiles.md](./model-profiles.md) | P12 MaaS 多模型 profile 与 C70 路由说明 |
| [traces.md](./traces.md) | P12 trace snapshot、脱敏与调试接口说明 |
| [api-errors.md](./api-errors.md) | P12 OpenAPI 错误响应矩阵与客户端兼容说明 |
```

更新 `docs/plans/03-agent/index.md`，增加 P12 主线；更新 `task-breakdown.md`，将 AG-P12-001 至 AG-P12-006 写入任务详情。

- [x] **Step 3: 总验证**

Run:

```powershell
go test ./...
go vet ./...
go build -o NUL ./cmd
.\scripts\smoke.ps1
.\scripts\release.ps1 -Version 0.1.0-local -Commit local-test -OutDir .\dist
```

Expected: 全部 PASS。若 release 产生 `dist/` 未跟踪产物，保留给人工检查或在收尾任务中加入 `.gitignore`。

## 风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| RBAC 与现有 admin token 语义冲突 | 旧脚本调用失败 | 默认缺失 `X-Role` 时按 `admin` 兼容本地开发，生产配置文档要求显式 role |
| 远端 registry 引入供应链风险 | 安装恶意技能 | 必须通过 SHA-256、A31 scanner、quarantine、X02 audit，不允许跳过扫描 |
| 多 profile 泄露 API key | 运维事故 | diagnostics、trace、OpenAPI 示例均不输出 API key，测试覆盖 |
| Trace 高基数字段膨胀 | 内存增长和隐私风险 | `MaxSpans` 环形保留，属性净化，不记录 prompt/tool input |
| OpenAPI 错误矩阵锁得过细 | 后续演进受限 | 只锁 status、ErrorResponse schema 和核心路径，不锁错误文案全文 |

## 完成定义

P12 完成时，Agent 应满足：

1. audit/quality 查询具备 role + company 边界，拒绝访问写入 X02 审计。
2. 远端 Skill registry 同步走 hash、扫描、quarantine、审计闭环。
3. MaaS C70 支持多 profile 配置和 CLI 显式选择，runtime 不依赖供应商细节。
4. `/debug/traces` 可导出脱敏 trace snapshot，并受 admin token 保护。
5. OpenAPI 包含稳定错误响应 schema、错误码矩阵和 golden 兼容测试。
6. 文档、CI、任务表同步，`go test ./...`、`go vet ./...`、`go build`、smoke、release 全部通过。
