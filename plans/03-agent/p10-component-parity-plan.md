---
id: "plans-agent-p10-component-parity-001"
title: "Agent P10 组件规范对齐计划"
type: "implementation-plan"
category: "plans/agent"
tags: ["plan", "agent", "p10", "component-parity", "architecture"]
version: "0.1.0"
created: "2026-05-15"
updated: "2026-05-15"
status: "draft"
related_docs:
  - path: "../../design/architecture/agent_components/index.md"
    relation: "derived_from"
  - path: "../../design/architecture/agent_components/agent-component-registry.md"
    relation: "derived_from"
  - path: "./task-breakdown.md"
    relation: "updates"
---

# Agent P10 Component Parity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 P2-P9 已完成的可运行 Agent 服务，进一步对齐 `agent_components` 设计规范中尚未完全展开的组件行为。

**Architecture:** P10 不再扩展运维外壳，而是回到 A00-A70 的组件规范本身。所有增强仍保持内部包边界清晰：上下文压缩留在 `internal/cognitive`，工具流水线留在 `internal/tool`，学习资产固化留在 `internal/evolution` 和 `internal/memory`，工作流高级原语留在 `internal/workflow`，跨组件契约放在 `internal/compat`。

**Tech Stack:** Go 1.26.0、Cobra、net/http、SQLite、slog、PowerShell、GitHub Actions。

---

## P10 定位

P2-P9 已完成“能运行、能服务化、能发布、能运维”。P10 的定位是 **component parity**：让实现从 MVP 行为向设计文档靠近，补齐当前最容易影响后续能力扩展的组件差距。

## 文件结构规划

| 路径 | 动作 | 职责 |
|------|------|------|
| `legion/legionAgent/internal/cognitive/compressor.go` | Create/Modify | A03 四层压缩策略、报告、checkpoint |
| `legion/legionAgent/internal/cognitive/core.go` | Modify | A00 接入更完整的 ContextCompressor 与系统提示缓存 |
| `legion/legionAgent/internal/tool/descriptor.go` | Create | A20 工具描述、schema、manifest |
| `legion/legionAgent/internal/tool/policy.go` | Modify/Create | A21 approval/sandbox/auto_allow 正交策略 |
| `legion/legionAgent/internal/tool/permission.go` | Modify/Create | A22 批量权限检查与角色覆盖 |
| `legion/legionAgent/internal/tool/guardrails.go` | Modify/Create | A23 timeout、重复失败、before/after hook |
| `legion/legionAgent/internal/evolution/distillation.go` | Modify/Create | A52 Gene 六元组、token 预算、refine |
| `legion/legionAgent/internal/evolution/solidify.go` | Modify/Create | A53 固化门控、资产封存、爆炸半径 |
| `legion/legionAgent/internal/evolution/eventlog.go` | Modify/Create | A54 immutable seal 与事件查询 |
| `legion/legionAgent/internal/workflow/definition.go` | Modify | A70 增加 loop/join/quorum DSL |
| `legion/legionAgent/internal/workflow/engine.go` | Modify | A70 loop/join/quorum 执行与 timeout |
| `legion/legionAgent/internal/server/http.go` | Modify | Workflow submit/resume API |
| `legion/legionAgent/internal/compat/component_parity_test.go` | Create | A/X/C 组件契约与设计对齐门禁 |
| `docs/agents/legion-agent/component-parity.md` | Create | P10 组件对齐说明 |

## P10 任务清单

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P10-001 | P0 | Registry/Compat | 组件实现对齐清单与 parity gate | `done` | `internal/compat`, `docs/agents/legion-agent` | A00-A70/X00-X05/C70 的实现状态、测试覆盖、降级策略可追踪 |
| AG-P10-002 | P0 | A00/A03/C70 | ContextCompressor 四层策略与 checkpoint | `done` | `internal/cognitive`, `internal/adapter` | trim/protect/summary/checkpoint 可测；C70 缺失时零 LLM 降级 |
| AG-P10-003 | P0 | A20-A23/X04/X05 | ToolRegistry 完整执行流水线 | `done` | `internal/tool`, `internal/port` | descriptor/schema/policy/permission/guardrails/timeout/sanitize 全链路可测 |
| AG-P10-004 | P1 | A30-A32 | Skill lifecycle 强化 | `done` | `internal/skill`, `internal/storage` | registry manifest、quarantine、install audit、disable/enable 可测 |
| AG-P10-005 | P0 | A52-A54/A43/X02 | Gene 蒸馏、固化与 sealed evolution log | `done` | `internal/evolution`, `internal/memory`, `internal/storage` | Gene 六元组、alpha 非空、版本寻址、immutable seal 可测 |
| AG-P10-006 | P1 | A60-A64/X00/X02 | 质量趋势与信任治理历史化 | `done` | `internal/quality`, `internal/storage`, `internal/observability` | eval/trust/degradation 历史可查，metrics/diagnostics 可见 |
| AG-P10-007 | P0 | A70/A10/A62/X00 | Workflow 高级原语与服务 API | `done` | `internal/workflow`, `internal/server`, `internal/storage` | loop/join/quorum/timeout，submit/resume/list API 可测 |
| AG-P10-008 | P1 | CI/Docs | P10 总验收与文档同步 | `done` | `.github/workflows`, `docs/agents/legion-agent`, `docs/plans/03-agent` | compat/parity/test/vet/build/smoke/release 全部通过 |

---

## Task 1: 组件实现对齐清单与 parity gate

**Files:**
- Create: `legion/legionAgent/internal/compat/component_parity_test.go`
- Create: `legion/legionAgent/internal/compat/testdata/component-parity.json`
- Create: `docs/agents/legion-agent/component-parity.md`

- [x] **Step 1: 写失败测试**

Run:

```powershell
go test ./internal/compat -run TestComponentParityManifest -count=1
```

Expected: FAIL，因为 `component-parity.json` 尚未存在。

- [x] **Step 2: 建立 parity manifest**

Manifest 至少包含：

```json
{
  "components": [
    {
      "id": "A03",
      "name": "ContextCompressor",
      "status": "planned",
      "packages": ["internal/cognitive"],
      "required_tests": ["TestContextCompressorFourLayerStrategy"],
      "degradation": "no_llm_trim_and_protect"
    }
  ]
}
```

- [x] **Step 3: 测试约束**

测试要求：
- 每个组件有 `id`、`name`、`status`、`packages`。
- `status` 只能是 `done`、`partial`、`planned`、`deferred`。
- P10 涉及组件必须出现在 manifest：`A03`、`A20`、`A21`、`A22`、`A23`、`A30`、`A31`、`A32`、`A52`、`A53`、`A54`、`A60`、`A61`、`A63`、`A64`、`A70`。

- [x] **Step 4: 文档**

`component-parity.md` 说明：
- P2-P9 已完成哪些组件 MVP。
- P10 补哪些 spec parity。
- 哪些组件仍然是 deferred。

## Task 2: ContextCompressor 四层策略与 checkpoint

**Files:**
- Create/Modify: `legion/legionAgent/internal/cognitive/compressor.go`
- Test: `legion/legionAgent/internal/cognitive/compressor_test.go`
- Modify: `legion/legionAgent/internal/cognitive/core.go`

- [x] **Step 1: 写失败测试**

Run:

```powershell
go test ./internal/cognitive -run "TestContextCompressorFourLayerStrategy|TestContextCompressorForceCheckpoint" -count=1
```

Expected: FAIL。

- [x] **Step 2: 数据类型**

实现：

```go
type CompressionStrategy int
type Message struct { Role, Kind, Content string; CreatedAt time.Time }
type MessageHistory struct { Messages []Message }
type CompressionReport struct { Strategy CompressionStrategy; LayersApplied []int; TokensBefore, TokensAfter int; Summary string; CompressedAt time.Time }
type CheckpointResult struct { CycleIndex int; Summary string; TokensSaved int }
```

- [x] **Step 3: 四层策略**

实现：
- Layer 1: old tool result trimming。
- Layer 2: head/tail protection。
- Layer 3: C70 auxiliary summary。
- Layer 4: force checkpoint。

验收：
- C70 为 nil 时，只执行 Layer 1/2。
- `ForceCheckpoint` 在 C70 可用时生成 checkpoint。
- report 记录 `LayersApplied` 和 token delta。

## Task 3: ToolRegistry 完整执行流水线

**Files:**
- Modify/Create: `legion/legionAgent/internal/tool/descriptor.go`
- Modify/Create: `legion/legionAgent/internal/tool/policy.go`
- Modify/Create: `legion/legionAgent/internal/tool/permission.go`
- Modify/Create: `legion/legionAgent/internal/tool/guardrails.go`
- Modify: `legion/legionAgent/internal/tool/registry.go`
- Test: `legion/legionAgent/internal/tool/*_test.go`

- [x] **Step 1: 写失败测试**

Run:

```powershell
go test ./internal/tool -run "TestToolRegistryPipeline|TestExecutionPolicyAutoAllow|TestToolGuardrailsTimeout" -count=1
```

Expected: FAIL。

- [x] **Step 2: ToolDescriptor 与 schema**

实现 tool descriptor：

```go
type Descriptor struct {
  Name string
  Description string
  InputSchema map[string]any
  RiskLevel string
  Timeout time.Duration
}
```

- [x] **Step 3: 执行流水线**

顺序固定：
1. resolve tool
2. schema validation
3. permission check
4. execution policy
5. guardrails before
6. execute with timeout
7. guardrails after
8. output sanitizer
9. audit log

验收：
- 权限拒绝不执行 handler。
- timeout 返回结构化失败。
- 输出走 sanitizer。

## Task 4: Skill lifecycle 强化

**Files:**
- Modify: `legion/legionAgent/internal/skill/system.go`
- Modify: `legion/legionAgent/internal/skill/installer.go`
- Modify: `legion/legionAgent/internal/storage/sqlite.go`
- Test: `legion/legionAgent/internal/skill/*_test.go`

- [x] **Step 1: 写失败测试**

Run:

```powershell
go test ./internal/skill -run "TestSkillInstallQuarantine|TestSkillEnableDisable|TestSkillInstallAudit" -count=1
```

Expected: FAIL。

- [x] **Step 2: lifecycle 状态**

状态：
- `candidate`
- `quarantined`
- `enabled`
- `disabled`
- `rejected`

- [x] **Step 3: 行为**

验收：
- scanner critical finding 自动 quarantine。
- enable 前必须扫描通过。
- install/enable/disable 写 audit event。
- CognitiveCore 只注入 enabled skill。

## Task 5: Gene 蒸馏、固化与 sealed evolution log

**Files:**
- Modify/Create: `legion/legionAgent/internal/evolution/distillation.go`
- Modify/Create: `legion/legionAgent/internal/evolution/solidify.go`
- Modify/Create: `legion/legionAgent/internal/evolution/eventlog.go`
- Modify: `legion/legionAgent/internal/memory/capability.go`
- Test: `legion/legionAgent/internal/evolution/*_test.go`

- [x] **Step 1: 写失败测试**

Run:

```powershell
go test ./internal/evolution -run "TestDistillationGeneSixTuple|TestSolidifyPipelineRequiresValidation|TestEvolutionEventSeal" -count=1
```

Expected: FAIL。

- [x] **Step 2: Gene 六元组**

实现字段：
- `m`
- `u`
- `pi`
- `alpha`
- `c`
- `v`

验收：
- `alpha` 强制非空。
- confidence 在 0~1。
- version 内容寻址。

- [x] **Step 3: Solidify gate**

固化要求：
- validation 通过。
- blast radius 低于阈值。
- 写 A54 event。
- 写 A43 capability asset。

## Task 6: 质量趋势与信任治理历史化

**Files:**
- Modify: `legion/legionAgent/internal/quality/eval.go`
- Modify: `legion/legionAgent/internal/quality/trust.go`
- Modify: `legion/legionAgent/internal/storage/sqlite.go`
- Modify: `legion/legionAgent/internal/observability/diagnostics.go`
- Test: `legion/legionAgent/internal/quality/*_test.go`

- [x] **Step 1: 写失败测试**

Run:

```powershell
go test ./internal/quality -run "TestEvalHistoryTrend|TestTrustScoreHistory|TestDiagnosticsIncludesQualitySummary" -count=1
```

Expected: FAIL。

- [x] **Step 2: 历史化**

新增存储：
- eval run history
- trust score snapshots
- degradation decisions

验收：
- 可按 agent/task/component 查询趋势。
- diagnostics 不暴露 prompt，只输出摘要。

## Task 7: Workflow 高级原语与服务 API

**Files:**
- Modify: `legion/legionAgent/internal/workflow/definition.go`
- Modify: `legion/legionAgent/internal/workflow/engine.go`
- Modify: `legion/legionAgent/internal/server/http.go`
- Modify: `legion/legionAgent/internal/storage/sqlite.go`
- Test: `legion/legionAgent/internal/workflow/engine_test.go`
- Test: `legion/legionAgent/internal/server/http_test.go`

- [x] **Step 1: 写失败测试**

Run:

```powershell
go test ./internal/workflow ./internal/server -run "TestWorkflowLoopJoinQuorum|TestHTTPWorkflowSubmitResume" -count=1
```

Expected: FAIL。

- [x] **Step 2: DSL**

新增/补齐：
- loop with max iterations
- join
- quorum failure policy
- node timeout

- [x] **Step 3: HTTP API**

新增：
- `POST /v1/workflows`
- `POST /v1/workflows/{id}/events`
- `GET /v1/workflows/{id}`

验收：
- workflow submit 写 storage。
- event resume 从 waiting_event 走到 completed。
- approval resume 仍由 A62 管理。

## Task 8: P10 总验收与文档同步

**Files:**
- Modify: `docs/agents/legion-agent/component-parity.md`
- Modify: `docs/agents/legion-agent/index.md`
- Modify: `docs/plans/03-agent/task-breakdown.md`
- Modify: `docs/plans/03-agent/index.md`
- Modify: `legion/legionAgent/.github/workflows/agent-ci.yml`

- [x] **Step 1: CI**

CI 追加：

```yaml
- name: Component parity
  run: go test ./internal/compat -run TestComponentParityManifest -count=1
```

- [x] **Step 2: 文档**

更新：
- P10 已完成组件。
- deferred 组件和原因。
- P11 建议。

- [x] **Step 3: 总验证**

Run:

```powershell
go test ./...
go vet ./...
go build -o NUL ./cmd
.\scripts\smoke.ps1
.\scripts\release.ps1 -Version 0.1.0-local -Commit local-test -OutDir .\dist
```

Expected: 全部 PASS。

## 风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| P10 范围过大 | 单轮实现拖长 | 严格按 AG-P10-001 到 AG-P10-008 分批推进 |
| ContextCompressor 引入 LLM 依赖 | 测试不稳定 | C70 使用 fake/recording client，nil 时降级 |
| 工具 schema 校验过重 | 依赖膨胀 | 先实现最小 JSON object required/type 校验 |
| Workflow 高级原语破坏现有 DSL | 兼容性回归 | 保留 P9 compat golden，新增 DSL golden |
| Gene 固化误写能力资产 | 质量回退 | 固化必须经过 validation gate 和 audit |

## 完成定义

P10 完成时，Agent 应满足：

1. 组件实现状态可以通过 parity manifest 自动检查。
2. A03 上下文压缩从阈值检测升级为四层策略。
3. A20-A23 工具执行具备 descriptor/schema/policy/permission/guardrails/timeout 全链路。
4. A52-A54 学习资产从 MVP 闭环升级为可验证、可封存、可审计的 Gene 固化。
5. A70 工作流支持 loop/join/quorum 和服务 API。
6. CI 同时验证 compat 和 component parity。
