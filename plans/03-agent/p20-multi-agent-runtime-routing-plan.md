---
id: "plans-agent-p20-routing-001"
title: "Agent P20 Multi-Agent Runtime Routing Implementation Plan"
type: "implementation-plan"
category: "plans/agent"
tags: ["plan", "agent", "multi-agent", "routing", "p20"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "plans-agent-wails-gui-impl-001"
    relation: "related_to"
    path: "./2026-06-24-wails-gui-plan.md"
---

# Agent P20 Multi-Agent Runtime Routing Implementation Plan

> 状态说明：P20 已在后续实现批次完成，当前权威状态见 [index.md](./index.md) 和 [task-breakdown.md](./task-breakdown.md)。本文后半部分保留早期实施检查清单，未勾选项作为历史计划记录，不再代表当前缺口。

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the L2 per-agent runtime routing described in `docs/agents/specs/2026-05-18-multi-agent-implementation-clarification.md`.

**Architecture:** Keep the existing single-agent runtime path as the fallback. Add a small `agentregistry` package for per-agent configuration, preserve `Task.AgentID` through scheduling, and let `Coordinator` resolve each task into the correct `domain.Agent`, MaaS client, context prefix, ToolRoot, and max tool rounds before calling `Runtime.RunTask`.

**Tech Stack:** Go 1.26, existing `internal/config`, `internal/contextfiles`, `internal/runtime`, `internal/task`, `internal/workflow`, `internal/service`, `internal/server`, SQLite/memory stores, Cobra CLI.

---

## Scope

This plan implements only L2: single-process per-agent runtime routing.

In scope:

- Preserve `TaskSpec.agent_id` from Workflow/API into Scheduler and Coordinator.
- Add `config.Config.Agents map[string]string`.
- Add `internal/agentregistry` for child agent config loading.
- Add Coordinator runtime resolution for registered agents.
- Wire `agent serve` with WorkflowEngine, WorkflowEvents, Coordinator heartbeat job, and AgentRegistry.
- Add examples and docs.

Out of scope:

- Full organization tree, department/team/report-chain permissions.
- Five-message communication protocol.
- Cross-process task distribution.
- TUI `delegate_task` tool.
- Agent hiring/budget/model-upgrade governance gates.

## File Structure

| File | Responsibility |
|------|----------------|
| `legion/legionAgent/internal/task/scheduler.go` | Preserve explicit `Task.AgentID`; only fill default agent when blank. |
| `legion/legionAgent/internal/task/scheduler_test.go` | Lock scheduler routing behavior. |
| `legion/legionAgent/internal/config/config.go` | Add `Agents map[string]string` root config field. |
| `legion/legionAgent/internal/config/config_test.go` | Verify `agents` map loads from JSON. |
| `legion/legionAgent/internal/agentregistry/config.go` | Define child `AgentConfig`. |
| `legion/legionAgent/internal/agentregistry/registry.go` | Load child configs relative to root config dir; expose `Get` and `Names`. |
| `legion/legionAgent/internal/agentregistry/registry_test.go` | Registry load, missing file, names copy, fallback lookup tests. |
| `legion/legionAgent/internal/runtime/coordinator.go` | Add optional per-agent resolver and fallback behavior. |
| `legion/legionAgent/internal/runtime/coordinator_test.go` | Verify registered agent routing, fallback, audit/event behavior. |
| `legion/legionAgent/internal/cli/command.go` | Build registry, workflow engine, coordinator heartbeat job in `agent serve`. |
| `legion/legionAgent/internal/cli/command_test.go` | Serve integration smoke for workflow engine availability and agent registry loading error. |
| `legion/legionAgent/configs/agent.full.example.json` | Add `agents` map example. |
| `legion/legionAgent/configs/agents/researcher.example.json` | Example researcher child config. |
| `legion/legionAgent/configs/agents/writer.example.json` | Example writer child config. |
| `docs/agents/legion-agent/configuration.md` | Document `agents` root config and child config shape. |
| `docs/agents/reference/multi-agent-collaboration.md` | Update status once serve + routing is implemented. |
| `docs/plans/03-agent/task-breakdown.md` | Track P20 tasks. |

## Task 1: Preserve Explicit Task AgentID

**Files:**

- Modify: `legion/legionAgent/internal/task/scheduler.go`
- Modify: `legion/legionAgent/internal/task/scheduler_test.go`

- [ ] **Step 1: Write failing scheduler test**

Add this test to `scheduler_test.go`:

```go
func TestSchedulerNextPreservesExplicitTaskAgentID(t *testing.T) {
	t.Parallel()

	scheduler := NewScheduler()
	task := domain.Task{
		ID:        "task-routed",
		CompanyID: "company-1",
		AgentID:   "researcher",
		Status:    domain.TaskPending,
		Input:     "research this",
	}
	if err := scheduler.Add(context.Background(), task); err != nil {
		t.Fatalf("Add(%q) error = %v, want nil", task.ID, err)
	}

	assigned, ok, err := scheduler.Next(context.Background(), "default-agent")
	if err != nil {
		t.Fatalf("Next(default-agent) error = %v, want nil", err)
	}
	if !ok {
		t.Fatalf("Next(default-agent) ok = false, want true")
	}
	if assigned.AgentID != "researcher" {
		t.Fatalf("Next(default-agent).AgentID = %q, want researcher", assigned.AgentID)
	}
}
```

- [ ] **Step 2: Run failing test**

Run:

```powershell
go test ./internal/task -run TestSchedulerNextPreservesExplicitTaskAgentID -count=1
```

Expected: FAIL because `Next` currently overwrites `AgentID`.

- [ ] **Step 3: Implement minimal scheduler fix**

Change `Scheduler.Next` assignment block to:

```go
if task.AgentID == "" {
	task.AgentID = agentID
}
task.Status = domain.TaskAssigned
```

- [ ] **Step 4: Verify task scheduler**

Run:

```powershell
go test ./internal/task -count=1
```

Expected: PASS.

## Task 2: Add Root Agents Config

**Files:**

- Modify: `legion/legionAgent/internal/config/config.go`
- Modify: `legion/legionAgent/internal/config/config_test.go`

- [ ] **Step 1: Add failing config test**

Add:

```go
func TestLoadAgentsConfig(t *testing.T) {
	t.Parallel()

	path := filepath.Join(t.TempDir(), "agent.json")
	body := `{
		"agents": {
			"researcher": "agents/researcher.json",
			"writer": "agents/writer.json"
		}
	}`
	if err := os.WriteFile(path, []byte(body), 0o600); err != nil {
		t.Fatalf("WriteFile(%q) error = %v, want nil", path, err)
	}

	cfg, err := Load(context.Background(), Options{Path: path})
	if err != nil {
		t.Fatalf("Load(%q) error = %v, want nil", path, err)
	}
	if cfg.Agents["researcher"] != "agents/researcher.json" {
		t.Fatalf("Load(%q).Agents[researcher] = %q, want agents/researcher.json", path, cfg.Agents["researcher"])
	}
	if cfg.Agents["writer"] != "agents/writer.json" {
		t.Fatalf("Load(%q).Agents[writer] = %q, want agents/writer.json", path, cfg.Agents["writer"])
	}
}
```

- [ ] **Step 2: Run failing config test**

Run:

```powershell
go test ./internal/config -run TestLoadAgentsConfig -count=1
```

Expected: FAIL because `Config.Agents` does not exist.

- [ ] **Step 3: Add config field and default**

Modify `Config`:

```go
type Config struct {
	Agents       map[string]string   `json:"agents"`
	Maas         MaasConfig          `json:"maas"`
	Storage      StorageConfig       `json:"storage"`
	Server       ServerConfig        `json:"server"`
	Service      ServiceConfig       `json:"service"`
	Runtime      RuntimeConfig       `json:"runtime"`
	TUI          TUIConfig           `json:"tui"`
	Skills       SkillsConfig        `json:"skills"`
	ContextFiles ContextFilesConfig  `json:"context_files"`
	Workspace    WorkspaceConfig     `json:"workspace"`
}
```

Add to `defaultConfig()`:

```go
Agents: map[string]string{},
```

- [ ] **Step 4: Verify config package**

Run:

```powershell
go test ./internal/config -count=1
```

Expected: PASS.

## Task 3: Create Agent Registry Package

**Files:**

- Create: `legion/legionAgent/internal/agentregistry/config.go`
- Create: `legion/legionAgent/internal/agentregistry/registry.go`
- Create: `legion/legionAgent/internal/agentregistry/registry_test.go`

- [ ] **Step 1: Write registry tests first**

Create `registry_test.go`:

```go
package agentregistry

import (
	"context"
	"errors"
	"os"
	"path/filepath"
	"reflect"
	"testing"

	"github.com/stardust/legion-agent/internal/config"
)

func TestLoadRegistryReadsChildAgentConfigsRelativeToConfigDir(t *testing.T) {
	t.Parallel()

	dir := t.TempDir()
	agentDir := filepath.Join(dir, "agents")
	if err := os.MkdirAll(agentDir, 0o700); err != nil {
		t.Fatalf("MkdirAll(%q) error = %v, want nil", agentDir, err)
	}
	writeAgentConfig(t, filepath.Join(agentDir, "researcher.json"), `{
		"id": "researcher",
		"role": "researcher",
		"maas_profile": "deep",
		"context_files": {
			"enabled": true,
			"root": ".",
			"soul_path": "configs/personas/researcher/SOUL.md",
			"tools_path": "configs/personas/researcher/TOOLS.md",
			"user_path": "configs/persona/USER.md",
			"memory_path": "configs/personas/researcher/MEMORY.md",
			"max_file_chars": 12000
		},
		"workspace": {"docs_root": "docs/research", "memory_root": "memory/researcher"},
		"skills": {"install_root": "skills/researcher"}
	}`)

	registry, err := Load(context.Background(), config.Config{
		Agents: map[string]string{"researcher": "agents/researcher.json"},
	}, dir)
	if err != nil {
		t.Fatalf("Load() error = %v, want nil", err)
	}
	got, ok := registry.Get("researcher")
	if !ok {
		t.Fatalf("Registry.Get(researcher) ok = false, want true")
	}
	if got.ID != "researcher" || got.Role != "researcher" || got.MaasProfile != "deep" {
		t.Fatalf("Registry.Get(researcher) = %#v, want id/role/profile loaded", got)
	}
	if got.ContextFiles.SoulPath != "configs/personas/researcher/SOUL.md" {
		t.Fatalf("AgentConfig.ContextFiles.SoulPath = %q, want researcher SOUL", got.ContextFiles.SoulPath)
	}
	if got.Workspace.MemoryRoot != "memory/researcher" {
		t.Fatalf("AgentConfig.Workspace.MemoryRoot = %q, want memory/researcher", got.Workspace.MemoryRoot)
	}
}

func TestLoadRegistryReturnsConfigNotFoundForMissingChild(t *testing.T) {
	t.Parallel()

	_, err := Load(context.Background(), config.Config{
		Agents: map[string]string{"missing": "agents/missing.json"},
	}, t.TempDir())
	if !errors.Is(err, ErrAgentConfigNotFound) {
		t.Fatalf("Load(missing child) error = %v, want ErrAgentConfigNotFound", err)
	}
}

func TestRegistryNamesReturnsSortedCopy(t *testing.T) {
	t.Parallel()

	registry := &Registry{agents: map[string]AgentConfig{
		"writer":     {ID: "writer"},
		"researcher": {ID: "researcher"},
	}}
	got := registry.Names()
	want := []string{"researcher", "writer"}
	if !reflect.DeepEqual(got, want) {
		t.Fatalf("Registry.Names() = %#v, want %#v", got, want)
	}
	got[0] = "mutated"
	gotAgain := registry.Names()
	if gotAgain[0] != "researcher" {
		t.Fatalf("Registry.Names() returned mutable backing slice, got %#v", gotAgain)
	}
}

func writeAgentConfig(t *testing.T, path string, body string) {
	t.Helper()
	if err := os.WriteFile(path, []byte(body), 0o600); err != nil {
		t.Fatalf("WriteFile(%q) error = %v, want nil", path, err)
	}
}
```

- [ ] **Step 2: Run failing agentregistry tests**

Run:

```powershell
go test ./internal/agentregistry -count=1
```

Expected: FAIL because package is not implemented.

- [ ] **Step 3: Add AgentConfig**

Create `config.go`:

```go
package agentregistry

import "github.com/stardust/legion-agent/internal/config"

type AgentConfig struct {
	ID           string                    `json:"id"`
	Role         string                    `json:"role"`
	MaasProfile  string                    `json:"maas_profile"`
	ContextFiles config.ContextFilesConfig `json:"context_files"`
	Workspace    config.WorkspaceConfig    `json:"workspace"`
	Skills       config.SkillsConfig       `json:"skills"`
}
```

- [ ] **Step 4: Add Registry loader**

Create `registry.go`:

```go
package agentregistry

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"os"
	"path/filepath"
	"sort"

	"github.com/stardust/legion-agent/internal/config"
)

var ErrAgentConfigNotFound = errors.New("agent config not found")

type Registry struct {
	agents map[string]AgentConfig
	root   config.Config
}

func Load(ctx context.Context, rootCfg config.Config, configDir string) (*Registry, error) {
	if err := ctx.Err(); err != nil {
		return nil, err
	}
	registry := &Registry{
		agents: make(map[string]AgentConfig, len(rootCfg.Agents)),
		root:   rootCfg,
	}
	for name, relPath := range rootCfg.Agents {
		agentCfg, err := loadAgentConfig(ctx, configDir, relPath)
		if err != nil {
			return nil, fmt.Errorf("load agent %q: %w", name, err)
		}
		if agentCfg.ID == "" {
			agentCfg.ID = name
		}
		registry.agents[name] = agentCfg
	}
	return registry, nil
}

func (r *Registry) Get(name string) (AgentConfig, bool) {
	if r == nil {
		return AgentConfig{}, false
	}
	agent, ok := r.agents[name]
	return agent, ok
}

func (r *Registry) Names() []string {
	if r == nil {
		return nil
	}
	names := make([]string, 0, len(r.agents))
	for name := range r.agents {
		names = append(names, name)
	}
	sort.Strings(names)
	return names
}

func loadAgentConfig(ctx context.Context, configDir string, relPath string) (AgentConfig, error) {
	if err := ctx.Err(); err != nil {
		return AgentConfig{}, err
	}
	path := relPath
	if !filepath.IsAbs(path) {
		path = filepath.Join(configDir, relPath)
	}
	data, err := os.ReadFile(path)
	if errors.Is(err, os.ErrNotExist) {
		return AgentConfig{}, fmt.Errorf("%w: %s", ErrAgentConfigNotFound, path)
	}
	if err != nil {
		return AgentConfig{}, fmt.Errorf("read agent config %q: %w", path, err)
	}
	var cfg AgentConfig
	if err := json.Unmarshal(data, &cfg); err != nil {
		return AgentConfig{}, fmt.Errorf("decode agent config %q: %w", path, err)
	}
	return cfg, nil
}
```

- [ ] **Step 5: Verify agentregistry**

Run:

```powershell
go test ./internal/agentregistry -count=1
```

Expected: PASS.

## Task 4: Add Coordinator Runtime Resolver Contract

**Files:**

- Modify: `legion/legionAgent/internal/runtime/coordinator.go`
- Modify: `legion/legionAgent/internal/runtime/coordinator_test.go`

- [ ] **Step 1: Write failing registered-agent routing test**

Add a controllable runtime double to `coordinator_test.go`:

```go
type recordingTaskRunner struct {
	agentID string
	result  string
}

func (r *recordingTaskRunner) RunTask(_ context.Context, agent domain.Agent, task domain.Task) (domain.TaskRun, error) {
	r.agentID = agent.ID
	return domain.TaskRun{
		ID:      task.ID + ":run",
		TaskID:  task.ID,
		AgentID: agent.ID,
		Result:  r.result,
	}, nil
}

type staticTaskRunnerResolver struct {
	agent domain.Agent
	runner *recordingTaskRunner
	ok     bool
}

func (r staticTaskRunnerResolver) ResolveTaskRunner(context.Context, domain.Task) (domain.Agent, TaskRunner, bool, error) {
	return r.agent, r.runner, r.ok, nil
}
```

Add test:

```go
func TestCoordinatorRoutesTaskToRegisteredAgentRunner(t *testing.T) {
	t.Parallel()

	scheduler := task.NewScheduler()
	if err := scheduler.Add(context.Background(), domain.Task{
		ID:        "task-research",
		CompanyID: "company-1",
		AgentID:   "researcher",
		Status:    domain.TaskPending,
		Input:     "research",
	}); err != nil {
		t.Fatalf("Add() error = %v, want nil", err)
	}
	audit := adapter.NewMemoryAuditLog()
	events := adapter.NewMemoryEventBus()
	researchRunner := &recordingTaskRunner{result: "research done"}
	coordinator := NewCoordinator(CoordinatorConfig{
		Agent: domain.Agent{ID: "default-agent", CompanyID: "company-1", Role: "developer", Status: domain.AgentActive},
		Scheduler: scheduler,
		Locks: task.NewLockStore(),
		Runtime: NewRuntime(Config{Maas: adapter.NewRecordingMaas("default result"), Audit: audit, Events: events}),
		TaskRunnerResolver: staticTaskRunnerResolver{
			agent:  domain.Agent{ID: "researcher", CompanyID: "company-1", Role: "researcher", Status: domain.AgentActive},
			runner: researchRunner,
			ok:     true,
		},
		Reviewer: quality.NewAegisReviewer(),
		Evaluator: quality.NewEvalEngine(3),
		Approvals: approval.NewService(),
		Audit: audit,
		Events: events,
		LockTTL: time.Minute,
	})

	result, ok, err := coordinator.Heartbeat(context.Background())
	if err != nil {
		t.Fatalf("Heartbeat() error = %v, want nil", err)
	}
	if !ok {
		t.Fatalf("Heartbeat() ok = false, want true")
	}
	if result.AgentID != "researcher" {
		t.Fatalf("Heartbeat() task AgentID = %q, want researcher", result.AgentID)
	}
	if researchRunner.agentID != "researcher" {
		t.Fatalf("registered runner agentID = %q, want researcher", researchRunner.agentID)
	}
}
```

- [ ] **Step 2: Run failing coordinator test**

Run:

```powershell
go test ./internal/runtime -run TestCoordinatorRoutesTaskToRegisteredAgentRunner -count=1
```

Expected: FAIL because `TaskRunnerResolver` and `TaskRunner` do not exist.

- [ ] **Step 3: Add TaskRunner and resolver interfaces**

In `coordinator.go`, add:

```go
type TaskRunner interface {
	RunTask(ctx context.Context, agent domain.Agent, task domain.Task) (domain.TaskRun, error)
}

type TaskRunnerResolver interface {
	ResolveTaskRunner(ctx context.Context, task domain.Task) (domain.Agent, TaskRunner, bool, error)
}
```

Change `CoordinatorConfig`:

```go
Runtime            TaskRunner
TaskRunnerResolver TaskRunnerResolver
```

Change `Coordinator` fields:

```go
runtime            TaskRunner
taskRunnerResolver TaskRunnerResolver
```

Change constructor to store `cfg.TaskRunnerResolver`.

- [ ] **Step 4: Add resolver selection in Heartbeat**

Before `RunTask`, add:

```go
agentToRun := c.agent
runner := c.runtime
if c.taskRunnerResolver != nil {
	resolvedAgent, resolvedRunner, ok, err := c.taskRunnerResolver.ResolveTaskRunner(ctx, taskToRun)
	if err != nil {
		_ = c.scheduler.Transition(ctx, taskToRun.ID, domain.TaskFailed)
		return domain.Task{}, false, fmt.Errorf("resolve task runner: %w", err)
	}
	if ok {
		agentToRun = resolvedAgent
		runner = resolvedRunner
	} else if taskToRun.AgentID != "" && taskToRun.AgentID != c.agent.ID {
		_ = c.publishEvent(ctx, taskToRun.ID, "agent_route_fallback", taskToRun.AgentID)
	}
}
run, err := runner.RunTask(ctx, agentToRun, taskToRun)
```

Add helper:

```go
func (c *Coordinator) publishEvent(ctx context.Context, taskID string, eventType string, message string) error {
	if c.events == nil {
		return nil
	}
	if err := c.events.Publish(ctx, domain.RuntimeEvent{
		Type:      eventType,
		TaskID:    taskID,
		Message:   message,
		CreatedAt: time.Now(),
	}); err != nil {
		return fmt.Errorf("publish %s event: %w", eventType, err)
	}
	return nil
}
```

- [ ] **Step 5: Verify coordinator package**

Run:

```powershell
go test ./internal/runtime -count=1
```

Expected: PASS.

## Task 5: Implement AgentRegistry Runtime Resolver

**Files:**

- Create: `legion/legionAgent/internal/runtime/agent_resolver.go`
- Create: `legion/legionAgent/internal/runtime/agent_resolver_test.go`

- [ ] **Step 1: Write resolver tests**

Create `agent_resolver_test.go`:

```go
package runtime

import (
	"context"
	"os"
	"path/filepath"
	"strings"
	"testing"

	"github.com/stardust/legion-agent/internal/adapter"
	"github.com/stardust/legion-agent/internal/agentregistry"
	"github.com/stardust/legion-agent/internal/config"
	"github.com/stardust/legion-agent/internal/domain"
)

func TestAgentRuntimeResolverBuildsPerAgentContextAndRuntime(t *testing.T) {
	t.Parallel()

	root := t.TempDir()
	writeFile(t, root, "AGENTS.md", "project rules")
	writeFile(t, root, "configs/personas/researcher/SOUL.md", "research soul")
	writeFile(t, root, "configs/personas/researcher/TOOLS.md", "research tools")
	writeFile(t, root, "configs/persona/USER.md", "user prefs")
	writeFile(t, root, "configs/personas/researcher/MEMORY.md", "research memory")

	registry := agentregistry.NewForTest(map[string]agentregistry.AgentConfig{
		"researcher": {
			ID: "researcher",
			Role: "researcher",
			MaasProfile: "deep",
			ContextFiles: config.ContextFilesConfig{
				Enabled: true,
				Root: root,
				AgentsPath: "AGENTS.md",
				SoulPath: "configs/personas/researcher/SOUL.md",
				ToolsPath: "configs/personas/researcher/TOOLS.md",
				UserPath: "configs/persona/USER.md",
				MemoryPath: "configs/personas/researcher/MEMORY.md",
				MaxFileChars: 20000,
			},
		},
	})
	resolver := NewAgentRuntimeResolver(AgentRuntimeResolverConfig{
		Registry: registry,
		RootConfig: config.Config{
			Maas: config.MaasConfig{
				DefaultProfile: "fast",
				Profiles: map[string]config.MaasProfile{
					"deep": {BaseURL: "https://deep.example.test", Model: "deep-model", APIKey: "deep-key"},
				},
			},
			Runtime: config.RuntimeConfig{MaxToolRounds: 4},
		},
		MaasFactory: func(profile string) (MaasRunnerFactoryResult, error) {
			if profile != "deep" {
				t.Fatalf("MaasFactory profile = %q, want deep", profile)
			}
			return MaasRunnerFactoryResult{Client: adapter.NewRecordingMaas("research result"), ModelName: "deep-model"}, nil
		},
	})

	agent, runner, ok, err := resolver.ResolveTaskRunner(context.Background(), domain.Task{ID: "task-1", AgentID: "researcher"})
	if err != nil {
		t.Fatalf("ResolveTaskRunner() error = %v, want nil", err)
	}
	if !ok {
		t.Fatalf("ResolveTaskRunner() ok = false, want true")
	}
	if agent.ID != "researcher" || agent.Role != "researcher" {
		t.Fatalf("ResolveTaskRunner() agent = %#v, want researcher role", agent)
	}
	run, err := runner.RunTask(context.Background(), agent, domain.Task{ID: "task-1", AgentID: "researcher", Input: "hello"})
	if err != nil {
		t.Fatalf("runner.RunTask() error = %v, want nil", err)
	}
	if !strings.Contains(run.Result, "research result") {
		t.Fatalf("runner.RunTask().Result = %q, want recording result", run.Result)
	}
}
```

Also add `NewForTest` helper in agentregistry during implementation:

```go
func NewForTest(agents map[string]AgentConfig) *Registry {
	copied := make(map[string]AgentConfig, len(agents))
	for name, cfg := range agents {
		copied[name] = cfg
	}
	return &Registry{agents: copied}
}
```

- [ ] **Step 2: Run failing resolver tests**

Run:

```powershell
go test ./internal/runtime -run TestAgentRuntimeResolverBuildsPerAgentContextAndRuntime -count=1
```

Expected: FAIL because resolver does not exist.

- [ ] **Step 3: Implement resolver config and MaaS factory result**

Create `agent_resolver.go` with:

```go
package runtime

import (
	"context"
	"fmt"

	"github.com/stardust/legion-agent/internal/agentregistry"
	"github.com/stardust/legion-agent/internal/cognitive"
	"github.com/stardust/legion-agent/internal/config"
	"github.com/stardust/legion-agent/internal/contextfiles"
	"github.com/stardust/legion-agent/internal/domain"
	"github.com/stardust/legion-agent/internal/port"
	"github.com/stardust/legion-agent/internal/tool"
)

type MaasRunnerFactoryResult struct {
	Client    port.MaasInferenceClient
	ModelName string
}

type MaasRunnerFactory func(profile string) (MaasRunnerFactoryResult, error)

type AgentRuntimeResolverConfig struct {
	Registry    *agentregistry.Registry
	RootConfig  config.Config
	Audit       port.AuditLog
	Events      port.EventBus
	MaasFactory MaasRunnerFactory
}

type AgentRuntimeResolver struct {
	registry    *agentregistry.Registry
	rootConfig  config.Config
	audit       port.AuditLog
	events      port.EventBus
	maasFactory MaasRunnerFactory
}

func NewAgentRuntimeResolver(cfg AgentRuntimeResolverConfig) *AgentRuntimeResolver {
	return &AgentRuntimeResolver{
		registry: cfg.Registry,
		rootConfig: cfg.RootConfig,
		audit: cfg.Audit,
		events: cfg.Events,
		maasFactory: cfg.MaasFactory,
	}
}
```

- [ ] **Step 4: Implement ResolveTaskRunner**

Add:

```go
func (r *AgentRuntimeResolver) ResolveTaskRunner(ctx context.Context, task domain.Task) (domain.Agent, TaskRunner, bool, error) {
	if r == nil || r.registry == nil || task.AgentID == "" {
		return domain.Agent{}, nil, false, nil
	}
	agentCfg, ok := r.registry.Get(task.AgentID)
	if !ok {
		return domain.Agent{}, nil, false, nil
	}
	maas, err := r.resolveMaas(agentCfg.MaasProfile)
	if err != nil {
		return domain.Agent{}, nil, false, err
	}
	contextPrefix, err := buildAgentContextPrefix(ctx, agentCfg, maas.ModelName)
	if err != nil {
		return domain.Agent{}, nil, false, err
	}
	contextBuilder := cognitive.NewCore(cognitive.NoopCompressor{}).WithContextFiles(contextPrefix)
	toolRoot := agentCfg.ContextFiles.Root
	if toolRoot == "" {
		toolRoot = r.rootConfig.ContextFiles.Root
	}
	rt := NewRuntime(Config{
		Maas:           maas.Client,
		Audit:          r.audit,
		Events:         r.events,
		ContextBuilder: contextBuilder,
		Tools:          tool.NewReadOnlyWorkspaceRegistry(toolRoot, r.audit),
		MaxToolRounds:  r.rootConfig.Runtime.MaxToolRounds,
	})
	return domain.Agent{
		ID:        agentCfg.ID,
		CompanyID: task.CompanyID,
		Role:      agentCfg.Role,
		Status:    domain.AgentActive,
	}, rt, true, nil
}
```

- [ ] **Step 5: Implement helpers**

Add:

```go
func (r *AgentRuntimeResolver) resolveMaas(profile string) (MaasRunnerFactoryResult, error) {
	if r.maasFactory == nil {
		return MaasRunnerFactoryResult{}, fmt.Errorf("maas factory is required for agent profile %q", profile)
	}
	return r.maasFactory(profile)
}

func buildAgentContextPrefix(ctx context.Context, agentCfg agentregistry.AgentConfig, modelName string) (string, error) {
	if !agentCfg.ContextFiles.Enabled {
		return "", nil
	}
	block, err := contextfiles.Load(ctx, contextfiles.Config{
		Enabled:      agentCfg.ContextFiles.Enabled,
		Root:         agentCfg.ContextFiles.Root,
		AgentsPath:   agentCfg.ContextFiles.AgentsPath,
		SoulPath:     agentCfg.ContextFiles.SoulPath,
		ToolsPath:    agentCfg.ContextFiles.ToolsPath,
		UserPath:     agentCfg.ContextFiles.UserPath,
		MemoryPath:   agentCfg.ContextFiles.MemoryPath,
		MaxFileChars: agentCfg.ContextFiles.MaxFileChars,
		ModelName:    modelName,
	})
	if err != nil {
		return "", err
	}
	return block.Render(), nil
}
```

- [ ] **Step 6: Verify runtime resolver**

Run:

```powershell
go test ./internal/agentregistry ./internal/runtime -count=1
```

Expected: PASS.

## Task 6: Wire Serve Workflow and Coordinator

**Files:**

- Modify: `legion/legionAgent/internal/cli/command.go`
- Modify: `legion/legionAgent/internal/cli/command_test.go`

- [ ] **Step 1: Add helper for config directory**

Add in `command.go`:

```go
func configDir(configPath string) string {
	if configPath == "" {
		return "."
	}
	return filepath.Dir(configPath)
}
```

- [ ] **Step 2: Add serve registry and MaaS wiring helpers**

Add registry helper:

```go
func loadServeAgentRegistry(ctx context.Context, cfg config.Config, configPath string) (*agentregistry.Registry, error) {
	return agentregistry.Load(ctx, cfg, configDir(configPath))
}
```

Add MaaS helper:

```go
func maasFactoryFromConfig(cfg config.MaasConfig) runtime.MaasRunnerFactory {
	return func(profile string) (runtime.MaasRunnerFactoryResult, error) {
		client, err := adapter.NewMaasClientFromProfile(cfg, profile)
		if err != nil {
			return runtime.MaasRunnerFactoryResult{}, err
		}
		display := tuiDisplayConfig(cfg, profile, "")
		return runtime.MaasRunnerFactoryResult{Client: client, ModelName: display.ModelName}, nil
	}
}
```

- [ ] **Step 3: Build live scheduler dependencies in serve**

Inside `newServeCommand`, after store creation, build a live in-process scheduler for runnable tasks. Keep the existing SQLite repository for workflow state/audit/quality persistence, but use this live scheduler for `/v1/tasks`, `WorkflowEngine`, and `Coordinator` during P20:

```go
liveTasks := task.NewScheduler()
workflowEvents := adapter.NewMemoryEventBus()
workflowEngine := workflow.NewEngine(workflow.Config{
	Scheduler: liveTasks,
	Approvals: approval.NewService(),
	Events: workflowEvents,
	Audit: auditLog,
})
```

Update `server.NewHTTPServer` wiring in the same function to use `liveTasks` as `Tasks`. This keeps the runtime queue consistent for tasks submitted by `/v1/tasks` and tasks produced by `/v1/workflows`.

- [ ] **Step 4: Load registry and build coordinator**

Add:

```go
registry, err := agentregistry.Load(cmd.Context(), cfg, configDir(configPath))
if err != nil {
	return err
}
resolver := runtime.NewAgentRuntimeResolver(runtime.AgentRuntimeResolverConfig{
	Registry: registry,
	RootConfig: cfg,
	Audit: auditLog,
	Events: workflowEvents,
	MaasFactory: maasFactoryFromConfig(cfg.Maas),
})
defaultMaas, err := adapter.NewMaasClientFromProfile(cfg.Maas, "")
if err != nil {
	return err
}
defaultContext, err := buildRunContextPrefix(cmd.Context(), cfg, false, tuiDisplayConfig(cfg.Maas, "", "").ModelName)
if err != nil {
	return err
}
coordinator := runtime.NewCoordinator(runtime.CoordinatorConfig{
	Agent: domain.Agent{ID: "default-agent", CompanyID: "default-company", Role: "developer", Status: domain.AgentActive},
	Scheduler: liveTasks,
	Locks: task.NewLockStore(),
	Runtime: runtime.NewRuntime(runtime.Config{
		Maas: defaultMaas,
		Audit: auditLog,
		Events: workflowEvents,
		ContextBuilder: cognitive.NewCore(cognitive.NoopCompressor{}).WithContextFiles(defaultContext),
		Tools: tool.NewReadOnlyWorkspaceRegistry(cfg.ContextFiles.Root, auditLog),
		MaxToolRounds: cfg.Runtime.MaxToolRounds,
	}),
	TaskRunnerResolver: resolver,
	Reviewer: quality.NewAegisReviewer(),
	Evaluator: quality.NewEvalEngine(3),
	Approvals: approval.NewService(),
	Audit: auditLog,
	Events: workflowEvents,
})
```

This step will require imports: `approval`, `agentregistry`, `cognitive`, `quality`, `runtime`, `tool`, `workflow`, and maybe `adapter` already exists.

- [ ] **Step 5: Register coordinator background job**

Create a background scheduler:

```go
background := task.NewBackgroundScheduler()
background.AddJob("agent-coordinator-heartbeat", func(ctx context.Context) error {
	_, _, err := coordinator.Heartbeat(ctx)
	return err
})
```

Pass it to `service.New`:

```go
Scheduler: background,
```

Pass workflow fields to HTTPServer:

```go
Tasks: liveTasks,
WorkflowEngine: workflowEngine,
WorkflowEvents: workflowEvents,
```

- [ ] **Step 6: Verify CLI package**

Run:

```powershell
go test ./internal/cli -count=1
```

Expected: PASS.

## Task 7: Serve Integration Tests

**Files:**

- Modify: `legion/legionAgent/internal/cli/command_test.go`
- Modify: `legion/legionAgent/internal/server/http_test.go` if direct HTTP tests need update.

- [ ] **Step 1: Add config registry load failure test**

Add test in CLI package:

```go
func TestServeReturnsErrorWhenConfiguredAgentFileMissing(t *testing.T) {
	t.Parallel()

	cfgPath := filepath.Join(t.TempDir(), "agent.json")
	body := `{
		"agents": {"researcher": "agents/missing.json"},
		"server": {"listen_addr": "127.0.0.1:0"}
	}`
	if err := os.WriteFile(cfgPath, []byte(body), 0o600); err != nil {
		t.Fatalf("WriteFile(%q) error = %v, want nil", cfgPath, err)
	}

	cfg, err := config.Load(context.Background(), config.Options{Path: cfgPath})
	if err != nil {
		t.Fatalf("Load(%q) error = %v, want nil", cfgPath, err)
	}
	_, err = loadServeAgentRegistry(context.Background(), cfg, cfgPath)
	if err == nil {
		t.Fatalf("loadServeAgentRegistry(missing child) error = nil, want non-nil")
	}
	if !strings.Contains(err.Error(), "agent config not found") {
		t.Fatalf("loadServeAgentRegistry(missing child) error = %v, want agent config not found", err)
	}
}
```

- [ ] **Step 2: Add workflow handler availability test**

Use `server.NewHTTPServer` existing tests as the pattern. Verify `POST /v1/workflows` no longer returns 503 when `agent serve` wiring helper is used.

Expected workflow request:

```json
{
  "id": "workflow-api",
  "root": {
    "id": "task-node",
    "kind": "agent_task",
    "task": {"id": "task-1", "agent_id": "researcher", "input": "research"}
  }
}
```

Expected response: `201 Created` or `202 Accepted`, not `503 Service Unavailable`.

- [ ] **Step 3: Run CLI/server focused tests**

Run:

```powershell
go test ./internal/cli ./internal/server -count=1
```

Expected: PASS.

## Task 8: Configuration Examples and Docs

**Files:**

- Modify: `legion/legionAgent/configs/agent.full.example.json`
- Create: `legion/legionAgent/configs/agents/researcher.example.json`
- Create: `legion/legionAgent/configs/agents/writer.example.json`
- Modify: `docs/agents/legion-agent/configuration.md`
- Modify: `docs/agents/reference/multi-agent-collaboration.md`

- [ ] **Step 1: Add root agents example**

Add near top of `agent.full.example.json`:

```json
"agents": {
  "_comment": "多 Agent 子配置映射。key 对应 workflow task.agent_id；value 是相对于本配置文件目录的子 agent 配置路径。",
  "researcher": "configs/agents/researcher.example.json",
  "writer": "configs/agents/writer.example.json"
},
```

JSON does not allow comments, so keep `_comment` as a key inside the object.

- [ ] **Step 2: Add researcher example**

Create `configs/agents/researcher.example.json`:

```json
{
  "id": "researcher",
  "role": "researcher",
  "maas_profile": "review",
  "context_files": {
    "enabled": true,
    "root": ".",
    "agents_path": "AGENTS.md",
    "soul_path": "configs/persona/SOUL.md",
    "tools_path": "configs/persona/TOOLS.md",
    "user_path": "configs/persona/USER.md",
    "memory_path": "configs/persona/MEMORY.md",
    "max_file_chars": 20000
  },
  "workspace": {
    "docs_root": "docs/research",
    "memory_root": "memory/researcher"
  },
  "skills": {
    "install_root": "skills/researcher"
  }
}
```

- [ ] **Step 3: Add writer example**

Create `configs/agents/writer.example.json`:

```json
{
  "id": "writer",
  "role": "writer",
  "maas_profile": "fast",
  "context_files": {
    "enabled": true,
    "root": ".",
    "agents_path": "AGENTS.md",
    "soul_path": "configs/persona/SOUL.md",
    "tools_path": "configs/persona/TOOLS.md",
    "user_path": "configs/persona/USER.md",
    "memory_path": "configs/persona/MEMORY.md",
    "max_file_chars": 20000
  },
  "workspace": {
    "docs_root": "docs/writing",
    "memory_root": "memory/writer"
  },
  "skills": {
    "install_root": "skills/writer"
  }
}
```

- [ ] **Step 4: Update configuration docs**

Add section:

```markdown
## 多 Agent 子配置

根配置的 `agents` 字段把 workflow `task.agent_id` 映射到子 agent 配置文件：

```json
{
  "agents": {
    "researcher": "configs/agents/researcher.example.json",
    "writer": "configs/agents/writer.example.json"
  }
}
```

子 agent 配置只描述差异项：`id`、`role`、`maas_profile`、`context_files`、`workspace`、`skills`。MaaS endpoint、storage、server、runtime max tool rounds 等共享项继承根配置。
```

- [ ] **Step 5: Update reference status**

Once code is complete, update `multi-agent-collaboration.md` status row to:

```markdown
| **Workflow Engine + per-agent runtime routing** | 在单进程内编排多个 task，并按 `task.agent_id` 切换 Agent 配置 | ✅ 已实现，推荐 |
```

## Task 9: Plan and Task Table Sync

**Files:**

- Modify: `docs/plans/03-agent/index.md`
- Modify: `docs/plans/03-agent/task-breakdown.md`

- [ ] **Step 1: Add P20 to index document list**

Add:

```markdown
| [p20-multi-agent-runtime-routing-plan.md](./p20-multi-agent-runtime-routing-plan.md) | P20 多 Agent Runtime 路由计划，补齐 AgentRegistry、Scheduler AgentID 保留、Coordinator per-agent Runtime 与 serve 工作流闭环 |
```

- [ ] **Step 2: Add P20 to phase table**

Add:

```markdown
| P20 multi-agent-runtime-routing | [p20-multi-agent-runtime-routing-plan.md](./p20-multi-agent-runtime-routing-plan.md) · [task-breakdown.md](./task-breakdown.md) | Workflow/A01/A00/C70/Config/Service |
```

- [x] **Step 3: Add status row**

Current state:

```markdown
| P20 | `done` | 多 Agent Runtime 路由：`task.agent_id` 可路由到不同 SOUL/MEMORY/model profile/workspace |
```

- [x] **Step 4: Add P20 task details**

Append:

```markdown
## P20 multi-agent-runtime-routing 任务详情

| 任务 ID | 优先级 | 组件 | 任务 | 状态 | 代码位置 | 验收 |
|---------|--------|------|------|------|----------|------|
| AG-P20-001 | P0 | Scheduler | 保留显式 Task.AgentID | `done` | `internal/task` | workflow task.agent_id 不被默认 coordinator 覆盖 |
| AG-P20-002 | P0 | Config/AgentRegistry | 根 agents 配置与子 agent loader | `done` | `internal/config`, `internal/agentregistry` | 可加载 researcher/writer 子配置，缺失文件报明确错误 |
| AG-P20-003 | P0 | Coordinator/A01 | per-agent runtime resolver | `done` | `internal/runtime` | 已注册 agent_id 使用对应 role/context/model/tool root |
| AG-P20-004 | P0 | CLI/Service/Workflow | serve 注入 workflow + coordinator 闭环 | `done` | `internal/cli`, `internal/service`, `internal/server` | POST workflow 后后台 heartbeat 可执行 task |
| AG-P20-005 | P1 | Docs/Examples | 示例配置与参考文档同步 | `done` | `configs`, `docs/agents`, `docs/plans` | 配置示例、协作参考、计划索引一致 |
```

## Task 10: Verification Gate

**Files:**

- No source changes.

- [ ] **Step 1: Run focused package tests**

Run:

```powershell
go test ./internal/task ./internal/config ./internal/agentregistry ./internal/runtime ./internal/cli ./internal/server -count=1
```

Expected: PASS.

- [ ] **Step 2: Run full test suite**

Run:

```powershell
go test ./...
```

Expected: PASS.

- [ ] **Step 3: Run vet**

Run:

```powershell
go vet ./...
```

Expected: no output and exit code 0.

- [ ] **Step 4: Run build**

Run:

```powershell
go build -o NUL ./cmd
```

Expected: binary build succeeds.

- [ ] **Step 5: Manual smoke**

Create temporary config with two child agents and submit:

```powershell
go run ./cmd -- serve --config .\agent.json --addr 127.0.0.1:8081
```

Then POST:

```powershell
curl -X POST http://127.0.0.1:8081/v1/workflows `
  -H "Authorization: Bearer change-me-admin-token" `
  -H "Content-Type: application/json" `
  -d '{"id":"multi-agent-smoke","root":{"id":"parallel-root","kind":"parallel","failure_policy":"collect_all","children":[{"id":"research-task","kind":"agent_task","task":{"id":"research-1","agent_id":"researcher","input":"调研多 Agent 实现现状"}},{"id":"writer-task","kind":"agent_task","task":{"id":"writer-1","agent_id":"writer","input":"整理调研结论"}}]}}}'
```

Expected:

- HTTP response is not 503.
- `/v1/tasks/research-1` keeps `agent_id=researcher`.
- `/v1/tasks/writer-1` keeps `agent_id=writer`.
- audit/events include task execution records.

## Self-Review

- Spec coverage: M1-M5 from the clarification document are covered by Tasks 1-9.
- Scope check: L3/L4 organization, communication protocol, and cross-process routing are explicitly out of scope.
- Placeholder scan: no deferred placeholder items are required for execution; every implementation task includes concrete files, code shape, and commands.
- Type consistency: plan uses `TaskRunner`, `TaskRunnerResolver`, `AgentRuntimeResolver`, and `AgentConfig` consistently after their defining tasks.
