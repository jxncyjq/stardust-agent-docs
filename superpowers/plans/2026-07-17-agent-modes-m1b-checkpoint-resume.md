---
title: Milestone 1b 实现计划 — 可挂起/可恢复工具循环 + 会话目录持久化 + 重启恢复
type: plan
status: ready
created: 2026-07-17
scope: legion/legionAgent（后端 runtime）
related:
  - "[[2026-07-15-agent-working-modes-design]]"
  - "[[2026-07-15-session-handoff-agent-modes]]"
  - "[[legion-git-repo-topology]]"
tags: [agent, runtime, checkpoint, suspend-resume, persistence, milestone-1b]
---

# Milestone 1b Implementation Plan — Suspendable/Resumable Tool Loop + Session Dir Persistence + Restart Recovery

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn `runtime.RunTask`'s synchronous tool loop into a checkpointable state machine that can persist mid-flight state to a per-session directory, release its goroutine on suspend, and resume from disk (including after a full process restart) to finish with the correct result.

**Architecture:** A new `internal/sessionstate` package owns (1) a single session-directory resolver (`workspace.root` + `~` expansion + fail-loud fallback) and (2) a `Checkpoint` disk store (`task-state.json`, atomic write, fail-loud on corruption). `runtime.Runtime` gains an injectable `ToolGate` seam (checked at each tool-round boundary) and a checkpoint store: when the gate says "suspend," RunTask serializes accumulated loop state and returns `ErrSuspended`; on entry it auto-loads any existing checkpoint and resumes. The `Coordinator` maps `ErrSuspended` to `TaskSuspended` (not `TaskFailed`). Restart recovery scans the store and re-registers suspended tasks. This is the M1b *mechanism*; the Manual-mode approval gate that will drive real suspends lands in M2 by implementing `ToolGate`.

**Tech Stack:** Go 1.26, standard library only (`encoding/json`, `os`, `path/filepath`), existing `internal/domain`, `internal/port`, `internal/task`, `internal/approval`, `internal/adapter` test doubles.

## Global Constraints

- **Fail-Loud 铁律** (legionAgent/CLAUDE.md §0): no fallback/zero-value masking, no silent `_ = err`, no silent `continue`/`return` on unexpected states. Corrupt checkpoint JSON, missing-but-required fields, disk write failures → return wrapped `error` (`fmt.Errorf("<action> <id>: %w", err)`); never swallow. The ONLY exception is contract-declared optional (checkpoint absent = legitimate "fresh task", handled via comma-ok).
- **Go 1.26** required (`go.mod`); GUI is a separate module and must not import server `internal/**`.
- **`-race` runs only in WSL** (Windows host has no gcc). Verification command for every task that adds/changes Go code under `internal/runtime`, `internal/task`, `internal/sessionstate`:
  ```bash
  wsl -d Ubuntu-22.04 -- bash -lc 'cd /mnt/f/source/stardust/Legion/legion/legionAgent && GOOS=linux GOARCH=amd64 CGO_ENABLED=1 CC=gcc $HOME/sdk/go/bin/go test -race ./internal/runtime/ ./internal/task/ ./internal/sessionstate/'
  ```
  Windows host runs plain `go test ./...` (go 1.26.0 in PATH) for non-race checks.
- **Completion gate** (CLAUDE.md 测试): `go build ./... && go vet ./... && go test ./...` all green, `gofmt -l .` empty for touched files.
- **Public API doc comments** required (Go doc style, start with identifier name).
- **Repo:** all work in `legion/legionAgent/` (git toplevel `F:\source\stardust\Legion\legion\legionAgent`, remote `jxncyjq-stardust-agent-server`, branch `master`). Confirm with `git rev-parse --show-toplevel` before committing. Push is gated — leave commits local unless the user authorizes push.
- **Pre-existing gofmt debt** in `cmd/gateway`, `internal/gateway/*`, `internal/domain/types.go` is NOT this feature's concern — do not reformat unrelated files; only `gofmt -w` files this plan touches.

---

## File Structure

**New package `internal/sessionstate`** (session-scoped disk state; single responsibility = "where session state lives + read/write task checkpoints"):
- `internal/sessionstate/resolver.go` — `ResolveWorkspaceRoot`, `SessionDir` (the single path resolver).
- `internal/sessionstate/resolver_test.go`
- `internal/sessionstate/checkpoint.go` — `Checkpoint`, `ToolEntrySnapshot`, `Store` (Save/Load/Delete/ListSuspended).
- `internal/sessionstate/checkpoint_test.go`

**Modified:**
- `internal/config/config.go` — add `WorkspaceConfig.Root` + default + env var.
- `internal/runtime/runtime.go` — add `ErrSuspended`, `ToolGate`, checkpoint fields to `Config`/`Runtime`; refactor `RunTask` into a resumable loop; auto-resume on entry; delete checkpoint on completion.
- `internal/runtime/checkpoint.go` (new file in runtime pkg) — `toolEntry`↔`ToolEntrySnapshot` conversion + checkpoint build/apply helpers (keeps `runtime.go` focused).
- `internal/runtime/coordinator.go` — map `ErrSuspended` → `TaskSuspended`.
- `internal/cli/command.go` — construct `sessionstate.Store` from `workspace.root`, inject into default runtime, call `RecoverSuspended` at startup.

**New tests:**
- `internal/runtime/checkpoint_test.go` — suspend writes checkpoint + `ErrSuspended`; resume completes.
- `internal/runtime/resume_e2e_test.go` — suspend → fresh Runtime+Store (simulated restart) → resume → correct result.
- `internal/runtime/coordinator_suspend_test.go` — coordinator lands suspended task in `TaskSuspended`.

---

## Task 1: `workspace.root` config field + session-directory resolver

**Files:**
- Create: `internal/sessionstate/resolver.go`
- Create: `internal/sessionstate/resolver_test.go`
- Modify: `internal/config/config.go` (WorkspaceConfig struct ~153-156; defaultConfig ~250-253; applyEnv ~361-366)

**Interfaces:**
- Produces:
  - `func ResolveWorkspaceRoot(configured string) (root string, warning string)` — expands `~`, returns configured dir if it exists, else `<home>/.stardust` with a non-empty `warning` string describing the fallback (empty `warning` = configured root used or empty-config default, no warning needed).
  - `func SessionDir(base, sessionKey string) string` — returns `filepath.Join(base, "session", sessionKey)`; `base` is the workspace root (M1b) or `<working_dir>/.stardust` (M3).
  - `config.WorkspaceConfig.Root string` (json `root`).

- [ ] **Step 1: Write the failing resolver test**

Create `internal/sessionstate/resolver_test.go`:

```go
package sessionstate

import (
	"os"
	"path/filepath"
	"strings"
	"testing"
)

func TestResolveWorkspaceRootUsesConfiguredExistingDir(t *testing.T) {
	dir := t.TempDir()
	root, warning := ResolveWorkspaceRoot(dir)
	if root != dir {
		t.Errorf("root = %q, want %q", root, dir)
	}
	if warning != "" {
		t.Errorf("warning = %q, want empty for a valid configured dir", warning)
	}
}

func TestResolveWorkspaceRootFallsBackAndWarnsOnMissingDir(t *testing.T) {
	missing := filepath.Join(t.TempDir(), "does-not-exist")
	root, warning := ResolveWorkspaceRoot(missing)
	home, err := os.UserHomeDir()
	if err != nil {
		t.Fatalf("UserHomeDir: %v", err)
	}
	want := filepath.Join(home, ".stardust")
	if root != want {
		t.Errorf("root = %q, want fallback %q", root, want)
	}
	if !strings.Contains(warning, missing) {
		t.Errorf("warning = %q, want it to mention the bad path %q", warning, missing)
	}
}

func TestResolveWorkspaceRootEmptyConfigUsesDefaultWithoutWarning(t *testing.T) {
	root, warning := ResolveWorkspaceRoot("")
	home, err := os.UserHomeDir()
	if err != nil {
		t.Fatalf("UserHomeDir: %v", err)
	}
	want := filepath.Join(home, ".stardust")
	if root != want {
		t.Errorf("root = %q, want default %q", root, want)
	}
	if warning != "" {
		t.Errorf("warning = %q, want empty when config is unset (default is not a misconfiguration)", warning)
	}
}

func TestResolveWorkspaceRootExpandsTilde(t *testing.T) {
	home, err := os.UserHomeDir()
	if err != nil {
		t.Fatalf("UserHomeDir: %v", err)
	}
	// ~/.stardust after expansion equals <home>/.stardust; whether it exists or
	// not the returned root must be the expanded absolute path, never literal "~".
	root, _ := ResolveWorkspaceRoot("~/.stardust")
	want := filepath.Join(home, ".stardust")
	if root != want {
		t.Errorf("root = %q, want expanded %q", root, want)
	}
}

func TestSessionDirJoinsUnderSessionSegment(t *testing.T) {
	got := SessionDir("/base", "sess-1")
	want := filepath.Join("/base", "session", "sess-1")
	if got != want {
		t.Errorf("SessionDir = %q, want %q", got, want)
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./internal/sessionstate/`
Expected: FAIL — build error, `undefined: ResolveWorkspaceRoot` / `undefined: SessionDir`.

- [ ] **Step 3: Write the resolver implementation**

Create `internal/sessionstate/resolver.go`:

```go
// Package sessionstate owns the on-disk home of per-session state: the single
// resolver that decides where a session's directory lives, and the checkpoint
// store that persists a suspended task's tool-loop state under it.
package sessionstate

import (
	"fmt"
	"os"
	"path/filepath"
	"strings"
)

// defaultRootName is the directory (under the user home) used when no valid
// workspace.root is configured. It reuses this repo's ".stardust" convention.
const defaultRootName = ".stardust"

// ResolveWorkspaceRoot turns the configured workspace.root into a concrete
// absolute directory. It expands a leading "~" to the user home. A configured,
// existing directory is used as-is. A configured-but-invalid path (non-empty but
// missing or not a directory) falls back to <home>/.stardust and returns a
// non-empty warning describing the fallback so the caller can log it fail-loud
// rather than silently swallowing a typo'd path. An empty configuration is the
// legitimate default (no warning).
func ResolveWorkspaceRoot(configured string) (root string, warning string) {
	home, err := os.UserHomeDir()
	if err != nil {
		// UserHomeDir failing is an unrecoverable environment fault; surface it
		// through the warning channel and fall back to a relative dir so the
		// caller still gets a usable path but is told something is wrong.
		return defaultRootName, fmt.Sprintf("cannot resolve user home dir: %v; using %q", err, defaultRootName)
	}
	fallback := filepath.Join(home, defaultRootName)

	trimmed := strings.TrimSpace(configured)
	if trimmed == "" {
		return fallback, ""
	}
	expanded := expandTilde(trimmed, home)
	info, statErr := os.Stat(expanded)
	if statErr == nil && info.IsDir() {
		return expanded, ""
	}
	return fallback, fmt.Sprintf("configured workspace.root %q not a dir, falling back to %q", expanded, fallback)
}

// expandTilde replaces a leading "~" (optionally "~/") with the user home dir.
// A "~" that is not the first path segment is left untouched.
func expandTilde(path, home string) string {
	if path == "~" {
		return home
	}
	if strings.HasPrefix(path, "~/") || strings.HasPrefix(path, `~\`) {
		return filepath.Join(home, path[2:])
	}
	return path
}

// SessionDir returns the directory that holds one session's persisted state:
// <base>/session/<sessionKey>. base is the workspace root (M1b) or, once
// working_dir lands (M3), <working_dir>/.stardust. sessionKey isolates state per
// session so concurrent tasks never write the same file.
func SessionDir(base, sessionKey string) string {
	return filepath.Join(base, "session", sessionKey)
}
```

- [ ] **Step 4: Run resolver test to verify it passes**

Run: `go test ./internal/sessionstate/`
Expected: PASS.

- [ ] **Step 5: Add `workspace.root` to config**

In `internal/config/config.go`, extend `WorkspaceConfig` (currently lines ~153-156):

```go
type WorkspaceConfig struct {
	// Root is the base directory for per-session state and workspace-relative
	// docs/memory. "~" is expanded; an unset/invalid value falls back to
	// <home>/.stardust (see sessionstate.ResolveWorkspaceRoot). DocsRoot and
	// MemoryRoot are resolved relative to it.
	Root       string `json:"root"`
	DocsRoot   string `json:"docs_root"`
	MemoryRoot string `json:"memory_root"`
}
```

In `defaultConfig()` update the `Workspace` literal (currently ~250-253):

```go
		Workspace: WorkspaceConfig{
			Root:       "~/.stardust",
			DocsRoot:   "docs",
			MemoryRoot: "memory",
		},
```

In `applyEnv`, add after the `LEGION_AGENT_MEMORY_ROOT` block (~361-366):

```go
	if value := os.Getenv("LEGION_AGENT_WORKSPACE_ROOT"); value != "" {
		cfg.Workspace.Root = value
	}
```

- [ ] **Step 6: Run config tests to verify nothing broke**

Run: `go test ./internal/config/`
Expected: PASS. (If `config_test.go` asserts the exact `WorkspaceConfig` default with a struct literal, update that expectation to include `Root: "~/.stardust"`.)

- [ ] **Step 7: Verify build + race**

Run (Windows host): `go build ./... && go vet ./...`
Run (WSL race): the Global Constraints `-race` command.
Expected: all green.

- [ ] **Step 8: Commit**

```bash
git add internal/sessionstate/resolver.go internal/sessionstate/resolver_test.go internal/config/config.go
git commit -m "feat(sessionstate): workspace.root config + session-dir resolver"
```

---

## Task 2: Checkpoint model + disk store

**Files:**
- Create: `internal/sessionstate/checkpoint.go`
- Create: `internal/sessionstate/checkpoint_test.go`

**Interfaces:**
- Consumes: `SessionDir` (Task 1), `domain.ToolCall` (`internal/domain`).
- Produces:
  - `const CheckpointSchemaVersion = 1`
  - `type ToolEntrySnapshot struct { Key string; Text string }`
  - `type Checkpoint struct { SchemaVersion int; TaskID, AgentID, SessionKey string; BasePrompt string; Round int; ToolEntries []ToolEntrySnapshot; PendingCalls []domain.ToolCall; PromptTokens, CompletionTokens, CachedTokens, TotalTokens int; Images []string; CreatedAt time.Time }`
  - `type Store struct { ... }`; `func NewStore(base string) *Store`
  - `func (s *Store) Save(cp Checkpoint) error`
  - `func (s *Store) Load(sessionKey string) (Checkpoint, bool, error)` — `(zero, false, nil)` when absent; `(zero, false, err)` on read/corrupt/version-mismatch (fail-loud).
  - `func (s *Store) Delete(sessionKey string) error`
  - `func (s *Store) ListSuspended() ([]Checkpoint, error)` — every `task-state.json` under `<base>/session/*/`.

Note: `SessionKey` = `domain.Task.SessionID` when non-empty, else `domain.Task.ID` (one-shot tasks with no session). The runtime picks the key; the store just uses it.

- [ ] **Step 1: Write the failing checkpoint store test**

Create `internal/sessionstate/checkpoint_test.go`:

```go
package sessionstate

import (
	"os"
	"path/filepath"
	"testing"
	"time"

	"github.com/stardust/legion-agent/internal/domain"
)

func sampleCheckpoint(key string) Checkpoint {
	return Checkpoint{
		SchemaVersion: CheckpointSchemaVersion,
		TaskID:        "task-1",
		AgentID:       "agent-1",
		SessionKey:    key,
		BasePrompt:    "system + task framing",
		Round:         2,
		ToolEntries:   []ToolEntrySnapshot{{Key: "read|path=a", Text: "- a success: hi"}},
		PendingCalls:  []domain.ToolCall{{ID: "c1", Name: "read", Arguments: map[string]string{"path": "b"}}},
		PromptTokens:  10,
		TotalTokens:   12,
		Images:        []string{"data:image/png;base64,xxx"},
		CreatedAt:     time.Unix(1_700_000_000, 0).UTC(),
	}
}

func TestStoreSaveLoadRoundTrip(t *testing.T) {
	store := NewStore(t.TempDir())
	cp := sampleCheckpoint("sess-1")
	if err := store.Save(cp); err != nil {
		t.Fatalf("Save: %v", err)
	}
	got, ok, err := store.Load("sess-1")
	if err != nil {
		t.Fatalf("Load: %v", err)
	}
	if !ok {
		t.Fatal("Load ok = false, want true after Save")
	}
	if got.Round != cp.Round || got.BasePrompt != cp.BasePrompt {
		t.Errorf("Load round/base = %d/%q, want %d/%q", got.Round, got.BasePrompt, cp.Round, cp.BasePrompt)
	}
	if len(got.PendingCalls) != 1 || got.PendingCalls[0].Name != "read" {
		t.Errorf("Load PendingCalls = %#v, want one read call", got.PendingCalls)
	}
	if len(got.ToolEntries) != 1 || got.ToolEntries[0].Key != "read|path=a" {
		t.Errorf("Load ToolEntries = %#v, want one entry", got.ToolEntries)
	}
}

func TestStoreLoadAbsentReturnsFalseNoError(t *testing.T) {
	store := NewStore(t.TempDir())
	_, ok, err := store.Load("nope")
	if err != nil {
		t.Fatalf("Load absent error = %v, want nil (absence is legitimate, not a fault)", err)
	}
	if ok {
		t.Fatal("Load absent ok = true, want false")
	}
}

func TestStoreLoadCorruptJSONFailsLoud(t *testing.T) {
	base := t.TempDir()
	store := NewStore(base)
	dir := SessionDir(base, "sess-bad")
	if err := os.MkdirAll(dir, 0o755); err != nil {
		t.Fatalf("mkdir: %v", err)
	}
	if err := os.WriteFile(filepath.Join(dir, checkpointFileName), []byte("{not json"), 0o644); err != nil {
		t.Fatalf("write: %v", err)
	}
	_, ok, err := store.Load("sess-bad")
	if err == nil {
		t.Fatal("Load corrupt error = nil, want fail-loud error")
	}
	if ok {
		t.Fatal("Load corrupt ok = true, want false")
	}
}

func TestStoreLoadVersionMismatchFailsLoud(t *testing.T) {
	base := t.TempDir()
	store := NewStore(base)
	cp := sampleCheckpoint("sess-v")
	cp.SchemaVersion = CheckpointSchemaVersion + 99
	// Save writes whatever version the checkpoint carries; Load must reject it.
	if err := store.Save(cp); err != nil {
		t.Fatalf("Save: %v", err)
	}
	_, _, err := store.Load("sess-v")
	if err == nil {
		t.Fatal("Load version-mismatch error = nil, want fail-loud error")
	}
}

func TestStoreDeleteRemovesCheckpoint(t *testing.T) {
	store := NewStore(t.TempDir())
	cp := sampleCheckpoint("sess-del")
	if err := store.Save(cp); err != nil {
		t.Fatalf("Save: %v", err)
	}
	if err := store.Delete("sess-del"); err != nil {
		t.Fatalf("Delete: %v", err)
	}
	_, ok, err := store.Load("sess-del")
	if err != nil {
		t.Fatalf("Load after delete: %v", err)
	}
	if ok {
		t.Fatal("Load after delete ok = true, want false")
	}
}

func TestStoreDeleteAbsentIsNoError(t *testing.T) {
	store := NewStore(t.TempDir())
	if err := store.Delete("never-existed"); err != nil {
		t.Fatalf("Delete absent = %v, want nil (idempotent)", err)
	}
}

func TestStoreListSuspendedReturnsAllCheckpoints(t *testing.T) {
	store := NewStore(t.TempDir())
	if err := store.Save(sampleCheckpoint("s1")); err != nil {
		t.Fatalf("Save s1: %v", err)
	}
	if err := store.Save(sampleCheckpoint("s2")); err != nil {
		t.Fatalf("Save s2: %v", err)
	}
	got, err := store.ListSuspended()
	if err != nil {
		t.Fatalf("ListSuspended: %v", err)
	}
	if len(got) != 2 {
		t.Fatalf("ListSuspended len = %d, want 2", len(got))
	}
}

func TestStoreListSuspendedEmptyIsEmptyNoError(t *testing.T) {
	store := NewStore(t.TempDir())
	got, err := store.ListSuspended()
	if err != nil {
		t.Fatalf("ListSuspended empty: %v", err)
	}
	if len(got) != 0 {
		t.Fatalf("ListSuspended empty len = %d, want 0", len(got))
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./internal/sessionstate/ -run TestStore`
Expected: FAIL — `undefined: NewStore`, `undefined: CheckpointSchemaVersion`, `undefined: checkpointFileName`.

- [ ] **Step 3: Write the checkpoint store implementation**

Create `internal/sessionstate/checkpoint.go`:

```go
package sessionstate

import (
	"encoding/json"
	"errors"
	"fmt"
	"os"
	"path/filepath"

	"github.com/stardust/legion-agent/internal/domain"
)

// CheckpointSchemaVersion versions the on-disk checkpoint format. Load rejects a
// checkpoint whose version it does not recognise (fail-loud) rather than
// half-decoding a future/older layout and resuming a task from wrong state.
const CheckpointSchemaVersion = 1

// checkpointFileName is the single per-session checkpoint file, per design §4.0.
const checkpointFileName = "task-state.json"

// ToolEntrySnapshot is the serialisable form of the runtime's internal toolEntry
// (whose fields are unexported). It preserves the dedup key and rendered text so
// a resumed loop rebuilds identical accumulated tool context.
type ToolEntrySnapshot struct {
	Key  string `json:"key"`
	Text string `json:"text"`
}

// Checkpoint is the serialised mid-flight state of a suspended tool loop: enough
// to re-enter RunTask and finish deterministically. It is written at a tool-round
// boundary — PendingCalls are the tool calls not yet executed when the runtime
// suspended.
type Checkpoint struct {
	SchemaVersion int                 `json:"schema_version"`
	TaskID        string              `json:"task_id"`
	AgentID       string              `json:"agent_id"`
	SessionKey    string              `json:"session_key"`
	BasePrompt    string              `json:"base_prompt"`
	Round         int                 `json:"round"`
	ToolEntries   []ToolEntrySnapshot `json:"tool_entries"`
	PendingCalls  []domain.ToolCall   `json:"pending_calls"`
	PromptTokens  int                 `json:"prompt_tokens"`
	CompletionTokens int              `json:"completion_tokens"`
	CachedTokens  int                 `json:"cached_tokens"`
	TotalTokens   int                 `json:"total_tokens"`
	Images        []string            `json:"images,omitempty"`
	CreatedAt     time.Time           `json:"created_at"`
}

// Store persists task checkpoints under a base directory, one file per session
// (SessionDir(base, key)/task-state.json).
type Store struct {
	base string
}

// NewStore returns a checkpoint store rooted at base (the resolved workspace
// root, or <working_dir>/.stardust once working_dir lands).
func NewStore(base string) *Store {
	return &Store{base: base}
}

// Save writes the checkpoint atomically (temp file + rename) so a crash mid-write
// never leaves a half-written task-state.json that Load would reject.
func (s *Store) Save(cp Checkpoint) error {
	if cp.SessionKey == "" {
		return errors.New("save checkpoint: empty SessionKey")
	}
	dir := SessionDir(s.base, cp.SessionKey)
	if err := os.MkdirAll(dir, 0o755); err != nil {
		return fmt.Errorf("create session dir %q: %w", dir, err)
	}
	data, err := json.MarshalIndent(cp, "", "  ")
	if err != nil {
		return fmt.Errorf("marshal checkpoint %s: %w", cp.SessionKey, err)
	}
	final := filepath.Join(dir, checkpointFileName)
	tmp := final + ".tmp"
	if err := os.WriteFile(tmp, data, 0o644); err != nil {
		return fmt.Errorf("write checkpoint tmp %q: %w", tmp, err)
	}
	if err := os.Rename(tmp, final); err != nil {
		return fmt.Errorf("rename checkpoint %q: %w", final, err)
	}
	return nil
}

// Load reads the checkpoint for sessionKey. Absence is legitimate (fresh task):
// it returns (zero, false, nil). Any other fault — unreadable file, corrupt JSON,
// or an unrecognised schema version — returns a fail-loud error rather than
// pretending the task has no prior state.
func (s *Store) Load(sessionKey string) (Checkpoint, bool, error) {
	path := filepath.Join(SessionDir(s.base, sessionKey), checkpointFileName)
	data, err := os.ReadFile(path)
	if errors.Is(err, os.ErrNotExist) {
		return Checkpoint{}, false, nil
	}
	if err != nil {
		return Checkpoint{}, false, fmt.Errorf("read checkpoint %q: %w", path, err)
	}
	var cp Checkpoint
	if err := json.Unmarshal(data, &cp); err != nil {
		return Checkpoint{}, false, fmt.Errorf("decode checkpoint %q: %w", path, err)
	}
	if cp.SchemaVersion != CheckpointSchemaVersion {
		return Checkpoint{}, false, fmt.Errorf("checkpoint %q schema version %d unsupported (want %d)", path, cp.SchemaVersion, CheckpointSchemaVersion)
	}
	return cp, true, nil
}

// Delete removes a session's checkpoint. A missing file is not an error (delete
// is idempotent — a completed or already-cleaned task is the normal case).
func (s *Store) Delete(sessionKey string) error {
	path := filepath.Join(SessionDir(s.base, sessionKey), checkpointFileName)
	if err := os.Remove(path); err != nil && !errors.Is(err, os.ErrNotExist) {
		return fmt.Errorf("remove checkpoint %q: %w", path, err)
	}
	return nil
}

// ListSuspended loads every checkpoint under <base>/session/*/task-state.json.
// A missing base dir yields an empty slice (no sessions yet). A corrupt or
// version-mismatched checkpoint fails loud — recovery must not silently skip a
// task it cannot restore.
func (s *Store) ListSuspended() ([]Checkpoint, error) {
	sessionsRoot := filepath.Join(s.base, "session")
	entries, err := os.ReadDir(sessionsRoot)
	if errors.Is(err, os.ErrNotExist) {
		return nil, nil
	}
	if err != nil {
		return nil, fmt.Errorf("read sessions root %q: %w", sessionsRoot, err)
	}
	var out []Checkpoint
	for _, entry := range entries {
		if !entry.IsDir() {
			continue
		}
		cp, ok, err := s.Load(entry.Name())
		if err != nil {
			return nil, fmt.Errorf("load suspended checkpoint for %q: %w", entry.Name(), err)
		}
		if !ok {
			continue // session dir without a checkpoint (e.g. only plans/) — legitimate
		}
		out = append(out, cp)
	}
	return out, nil
}
```

Add `"time"` to the import block (the `Checkpoint.CreatedAt time.Time` field needs it):

```go
import (
	"encoding/json"
	"errors"
	"fmt"
	"os"
	"path/filepath"
	"time"

	"github.com/stardust/legion-agent/internal/domain"
)
```

- [ ] **Step 4: Run checkpoint test to verify it passes**

Run: `go test ./internal/sessionstate/`
Expected: PASS (all Task 1 + Task 2 tests).

- [ ] **Step 5: Verify build + vet + race**

Run (Windows): `go build ./... && go vet ./...`
Run (WSL): the `-race` command.
Expected: green.

- [ ] **Step 6: Commit**

```bash
git add internal/sessionstate/checkpoint.go internal/sessionstate/checkpoint_test.go
git commit -m "feat(sessionstate): task checkpoint disk store (atomic write, fail-loud load)"
```

---

## Task 3: Runtime checkpoint plumbing + `ToolGate` seam (no behaviour change)

This task adds the fields, types, and conversion helpers WITHOUT changing RunTask's behaviour yet (gate nil + store nil → identical to today). Task 4 wires them into the loop.

**Files:**
- Modify: `internal/runtime/runtime.go` (Config struct ~30-59; Runtime struct ~70-88; NewRuntime ~90-124; add `ErrSuspended` ~19-22)
- Create: `internal/runtime/checkpoint.go`
- Create: `internal/runtime/checkpoint_helpers_test.go`

**Interfaces:**
- Consumes: `sessionstate.Store`, `sessionstate.Checkpoint`, `sessionstate.ToolEntrySnapshot` (Task 2).
- Produces:
  - `var ErrSuspended = errors.New("runtime suspended pending decision")`
  - `type ToolGate interface { ShouldSuspend(ctx context.Context, task domain.Task, calls []domain.ToolCall) (bool, error) }`
  - `Config.Checkpoints *sessionstate.Store`, `Config.ToolGate ToolGate`
  - `func sessionKeyForTask(task domain.Task) string` — `task.SessionID` if non-empty else `task.ID`.
  - `func snapshotToolEntries([]toolEntry) []sessionstate.ToolEntrySnapshot` and `func restoreToolEntries([]sessionstate.ToolEntrySnapshot) []toolEntry`.

- [ ] **Step 1: Write the failing helper test**

Create `internal/runtime/checkpoint_helpers_test.go`:

```go
package runtime

import (
	"testing"

	"github.com/stardust/legion-agent/internal/domain"
)

func TestSessionKeyForTaskPrefersSessionID(t *testing.T) {
	if got := sessionKeyForTask(domain.Task{ID: "t1", SessionID: "s1"}); got != "s1" {
		t.Errorf("sessionKeyForTask = %q, want s1", got)
	}
	if got := sessionKeyForTask(domain.Task{ID: "t1"}); got != "t1" {
		t.Errorf("sessionKeyForTask (no session) = %q, want t1", got)
	}
}

func TestToolEntrySnapshotRoundTrip(t *testing.T) {
	entries := []toolEntry{{key: "read|path=a", text: "- a success: hi"}, {key: "list", text: "- list success: x"}}
	restored := restoreToolEntries(snapshotToolEntries(entries))
	if len(restored) != len(entries) {
		t.Fatalf("restored len = %d, want %d", len(restored), len(entries))
	}
	for i := range entries {
		if restored[i].key != entries[i].key || restored[i].text != entries[i].text {
			t.Errorf("restored[%d] = %+v, want %+v", i, restored[i], entries[i])
		}
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./internal/runtime/ -run 'TestSessionKeyForTask|TestToolEntrySnapshot'`
Expected: FAIL — `undefined: sessionKeyForTask`, `undefined: snapshotToolEntries`.

- [ ] **Step 3: Add `ErrSuspended` and `ToolGate`, extend Config/Runtime**

In `internal/runtime/runtime.go`, extend the error block (~19-22):

```go
var (
	ErrInterrupted     = errors.New("runtime interrupted")
	ErrMaasUnavailable = errors.New("maas inference client unavailable")
	// ErrSuspended is returned by RunTask when the ToolGate pauses execution at a
	// tool-round boundary. The runtime has already written a checkpoint; the
	// coordinator maps this to TaskSuspended (not TaskFailed) and the goroutine
	// is released. A later run (this process or after restart) auto-resumes.
	ErrSuspended = errors.New("runtime suspended pending decision")
)
```

Add the `ToolGate` interface near `ContextBuilder` (~26-28):

```go
// ToolGate decides, at each tool-round boundary, whether the runtime must
// suspend before executing the given pending tool calls (e.g. awaiting human
// approval in Manual mode). A nil gate never suspends — Auto behaviour. M1b ships
// only the seam; the approval-backed implementation lands in M2.
type ToolGate interface {
	ShouldSuspend(ctx context.Context, task domain.Task, calls []domain.ToolCall) (bool, error)
}
```

Add to `Config` struct (after the `MaxConcurrent int` field, ~58):

```go
	// Checkpoints persists suspended tool-loop state so a task can resume after
	// its goroutine is released (and after a process restart). Nil disables
	// suspend/resume (the loop runs straight through, legacy behaviour).
	Checkpoints *sessionstate.Store
	// ToolGate gates each tool round for suspension. Nil never suspends.
	ToolGate ToolGate
```

Add to `Runtime` struct (after `subTaskSeq`, ~87):

```go
	checkpoints *sessionstate.Store
	toolGate    ToolGate
```

In `NewRuntime`, set them in the returned `&Runtime{...}` literal (after `maxConcurrent:`):

```go
		checkpoints:        cfg.Checkpoints,
		toolGate:           cfg.ToolGate,
```

Add the `sessionstate` import to `runtime.go`'s import block:

```go
	"github.com/stardust/legion-agent/internal/sessionstate"
```

- [ ] **Step 4: Write the conversion + key helpers**

Create `internal/runtime/checkpoint.go`:

```go
package runtime

import (
	"github.com/stardust/legion-agent/internal/domain"
	"github.com/stardust/legion-agent/internal/sessionstate"
)

// sessionKeyForTask picks the directory key for a task's persisted state: its
// session id when it has one, otherwise its task id (one-shot tasks with no
// session still get an isolated checkpoint dir).
func sessionKeyForTask(task domain.Task) string {
	if task.SessionID != "" {
		return task.SessionID
	}
	return task.ID
}

// snapshotToolEntries converts the runtime's internal (unexported-field) tool
// context into the serialisable snapshot form for a checkpoint.
func snapshotToolEntries(entries []toolEntry) []sessionstate.ToolEntrySnapshot {
	out := make([]sessionstate.ToolEntrySnapshot, 0, len(entries))
	for _, e := range entries {
		out = append(out, sessionstate.ToolEntrySnapshot{Key: e.key, Text: e.text})
	}
	return out
}

// restoreToolEntries rebuilds internal tool context from a checkpoint snapshot,
// so a resumed loop re-accumulates identical deduplicated context.
func restoreToolEntries(snaps []sessionstate.ToolEntrySnapshot) []toolEntry {
	out := make([]toolEntry, 0, len(snaps))
	for _, s := range snaps {
		out = append(out, toolEntry{key: s.Key, text: s.Text})
	}
	return out
}
```

- [ ] **Step 5: Run helper test + full runtime suite to verify pass + no regressions**

Run: `go test ./internal/runtime/`
Expected: PASS (helpers pass; existing tests unaffected — gate/store are nil everywhere).

- [ ] **Step 6: Verify build + vet + race**

Run (Windows): `go build ./... && go vet ./...`
Run (WSL): the `-race` command.
Expected: green.

- [ ] **Step 7: Commit**

```bash
git add internal/runtime/runtime.go internal/runtime/checkpoint.go internal/runtime/checkpoint_helpers_test.go
git commit -m "feat(runtime): checkpoint store + ToolGate seam plumbing (no behavior change)"
```

---

## Task 4: Resumable tool loop — suspend writes checkpoint, entry auto-resumes

This is the core refactor. Extract the tool loop so it (a) starts fresh, (b) checks the gate before executing each round's calls and suspends (checkpoint + `ErrSuspended`) when told, and (c) can resume from a loaded checkpoint. On successful completion the checkpoint is deleted.

**Files:**
- Modify: `internal/runtime/runtime.go` (`RunTask` ~144-284 — split into entry + `runToolLoop`)
- Create: `internal/runtime/checkpoint_test.go`

**Interfaces:**
- Consumes: `ErrSuspended`, `ToolGate`, `sessionKeyForTask`, `snapshotToolEntries`, `restoreToolEntries`, `sessionstate.Checkpoint` (Task 3).
- Produces (internal): `loopState` struct carrying `{basePrompt string; round int; toolCtx []toolEntry; resp port.InferenceResponse; promptTokens, completionTokens, cachedTokens, totalTokens int}`; `func (r *Runtime) runToolLoop(ctx, requestID, agent, task, st loopState) (domain.TaskRun, error)`.

**Design (round-boundary suspend):** In `runToolLoop`, at the top of each round — before `executeToolCalls` — if `r.toolGate != nil` and `r.checkpoints != nil`, call `r.toolGate.ShouldSuspend(ctx, task, st.resp.ToolCalls)`. If true: build a `Checkpoint{Round: st.round, ToolEntries: snapshot(st.toolCtx), PendingCalls: st.resp.ToolCalls, tokens…, BasePrompt: st.basePrompt, Images: task.Images, SchemaVersion: CheckpointSchemaVersion, SessionKey: sessionKeyForTask(task), TaskID, AgentID}`, `Save` it, and return `ErrSuspended`. On resume, `st.resp.ToolCalls` is set to `cp.PendingCalls` and the loop executes them exactly as a fresh round would.

- [ ] **Step 1: Write the failing suspend/resume unit test**

Create `internal/runtime/checkpoint_test.go`:

```go
package runtime

import (
	"context"
	"errors"
	"sync"
	"testing"

	"github.com/stardust/legion-agent/internal/adapter"
	"github.com/stardust/legion-agent/internal/domain"
	"github.com/stardust/legion-agent/internal/port"
	"github.com/stardust/legion-agent/internal/sessionstate"
	"github.com/stardust/legion-agent/internal/tool"
)

// scriptedMaas returns a tool call on its first inference and a final text answer
// afterwards, so a single tool round is exercised.
type scriptedMaas struct {
	mu    sync.Mutex
	calls int
}

func (m *scriptedMaas) Generate(ctx context.Context, req port.InferenceRequest) (port.InferenceResponse, error) {
	if err := ctx.Err(); err != nil {
		return port.InferenceResponse{}, err
	}
	m.mu.Lock()
	defer m.mu.Unlock()
	m.calls++
	if m.calls == 1 {
		return port.InferenceResponse{ToolCalls: []domain.ToolCall{{
			ID:        "call-1",
			Name:      "echo",
			Arguments: map[string]string{"text": "hi"},
		}}}, nil
	}
	return port.InferenceResponse{Text: "final answer"}, nil
}

// gateOnce suspends exactly once, then allows — modelling "decision arrived".
type gateOnce struct {
	mu        sync.Mutex
	suspended bool
}

func (g *gateOnce) ShouldSuspend(_ context.Context, _ domain.Task, _ []domain.ToolCall) (bool, error) {
	g.mu.Lock()
	defer g.mu.Unlock()
	if !g.suspended {
		g.suspended = true
		return true, nil
	}
	return false, nil
}

// echoTool is a trivial registered tool so executeToolCalls has something real
// to run on resume.
func echoRegistry(t *testing.T) *tool.Registry {
	t.Helper()
	reg := tool.NewRegistry()
	if err := reg.Register(tool.Definition{
		Name:        "echo",
		Description: "echo text back",
		InputSchema: map[string]any{"type": "object", "properties": map[string]any{"text": map[string]any{"type": "string"}}},
		Handler: func(_ context.Context, _ domain.Agent, call domain.ToolCall) (domain.ToolResult, error) {
			return domain.ToolResult{CallID: call.ID, Success: true, Output: call.Arguments["text"]}, nil
		},
	}); err != nil {
		t.Fatalf("register echo: %v", err)
	}
	return reg
}

func TestRunTaskSuspendsAndWritesCheckpoint(t *testing.T) {
	store := sessionstate.NewStore(t.TempDir())
	runner := NewRuntime(Config{
		Maas:        &scriptedMaas{},
		Audit:       adapter.NewMemoryAuditLog(),
		Events:      adapter.NewMemoryEventBus(),
		Tools:       echoRegistry(t),
		Checkpoints: store,
		ToolGate:    &gateOnce{},
	})
	task := domain.Task{ID: "task-1", SessionID: "sess-1", AgentID: "agent-1", Status: domain.TaskRunning, Input: "go"}

	_, err := runner.RunTask(context.Background(), domain.Agent{ID: "agent-1"}, task)
	if !errors.Is(err, ErrSuspended) {
		t.Fatalf("RunTask err = %v, want ErrSuspended", err)
	}
	cp, ok, err := store.Load("sess-1")
	if err != nil {
		t.Fatalf("Load checkpoint: %v", err)
	}
	if !ok {
		t.Fatal("no checkpoint written on suspend")
	}
	if len(cp.PendingCalls) != 1 || cp.PendingCalls[0].Name != "echo" {
		t.Errorf("checkpoint PendingCalls = %#v, want one echo call", cp.PendingCalls)
	}
}

func TestRunTaskResumesFromCheckpointToCompletion(t *testing.T) {
	store := sessionstate.NewStore(t.TempDir())
	maas := &scriptedMaas{}
	gate := &gateOnce{}
	cfg := Config{
		Maas:        maas,
		Audit:       adapter.NewMemoryAuditLog(),
		Events:      adapter.NewMemoryEventBus(),
		Tools:       echoRegistry(t),
		Checkpoints: store,
		ToolGate:    gate,
	}
	runner := NewRuntime(cfg)
	task := domain.Task{ID: "task-1", SessionID: "sess-1", AgentID: "agent-1", Status: domain.TaskRunning, Input: "go"}

	// First run suspends.
	if _, err := runner.RunTask(context.Background(), domain.Agent{ID: "agent-1"}, task); !errors.Is(err, ErrSuspended) {
		t.Fatalf("first run err = %v, want ErrSuspended", err)
	}
	// Second run (same runtime, gate now allows) auto-resumes from the checkpoint.
	run, err := runner.RunTask(context.Background(), domain.Agent{ID: "agent-1"}, task)
	if err != nil {
		t.Fatalf("resume run err = %v, want nil", err)
	}
	if run.Result != "final answer" {
		t.Errorf("resume result = %q, want %q", run.Result, "final answer")
	}
	// Checkpoint deleted after successful completion.
	_, ok, err := store.Load("sess-1")
	if err != nil {
		t.Fatalf("Load after completion: %v", err)
	}
	if ok {
		t.Error("checkpoint still present after successful completion, want deleted")
	}
}
```

> Before implementing, confirm the tool registry constructor/registration API names used above (`tool.NewRegistry`, `tool.Definition`, `reg.Register`, `Handler` signature) against `internal/tool/`. If they differ, adjust the test helper to the real API — the behaviour asserted (a registered `echo` tool returning its `text` arg) stays the same.

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./internal/runtime/ -run 'TestRunTaskSuspends|TestRunTaskResumes'`
Expected: FAIL — RunTask never returns `ErrSuspended` (loop ignores the gate).

- [ ] **Step 3: Refactor `RunTask` into entry + `runToolLoop`**

Replace the body of `RunTask` (lines ~144-284) so it: publishes `task_started`, checks interrupt/maas, then **auto-resumes** if a checkpoint exists, else builds the prompt + does the initial generate and enters `runToolLoop`. Extract everything from the `var toolCtx []toolEntry` declaration through the end into `runToolLoop`.

New `RunTask`:

```go
func (r *Runtime) RunTask(ctx context.Context, agent domain.Agent, task domain.Task) (domain.TaskRun, error) {
	started := time.Now()
	requestID := task.ID + ":run"
	if err := r.events.Publish(ctx, domain.RuntimeEvent{
		Type:      "task_started",
		TaskID:    task.ID,
		Message:   "runtime started",
		CreatedAt: started,
	}); err != nil {
		return domain.TaskRun{}, fmt.Errorf("publish task started event: %w", err)
	}
	if r.interrupted.Load() {
		_ = r.publishLearning(ctx, agent, task, evolution.SignalFailure, evolution.FailureReasonInterrupted, true)
		return domain.TaskRun{}, ErrInterrupted
	}
	if r.maas == nil {
		_ = r.publishLearning(ctx, agent, task, evolution.SignalFailure, evolution.FailureReasonInferenceError, true)
		return domain.TaskRun{}, ErrMaasUnavailable
	}

	// Resume path: a persisted checkpoint means this task previously suspended.
	// Rebuild loop state from disk and re-enter the loop with the pending calls,
	// skipping the initial prompt build + generate.
	if r.checkpoints != nil {
		cp, ok, err := r.checkpoints.Load(sessionKeyForTask(task))
		if err != nil {
			return domain.TaskRun{}, fmt.Errorf("load checkpoint for task %s: %w", task.ID, err)
		}
		if ok {
			st := loopState{
				started:          started,
				basePrompt:       cp.BasePrompt,
				round:            cp.Round,
				toolCtx:          restoreToolEntries(cp.ToolEntries),
				resp:             port.InferenceResponse{ToolCalls: cp.PendingCalls},
				promptTokens:     cp.PromptTokens,
				completionTokens: cp.CompletionTokens,
				cachedTokens:     cp.CachedTokens,
				totalTokens:      cp.TotalTokens,
			}
			return r.runToolLoop(ctx, requestID, agent, task, st)
		}
	}

	prompt, err := r.buildPrompt(ctx, agent, task)
	if err != nil {
		return domain.TaskRun{}, err
	}
	basePrompt := prompt
	resp, err := r.generate(ctx, requestID, prompt, task.Images, len([]rune(basePrompt)))
	if err != nil {
		_ = r.publishLearning(ctx, agent, task, evolution.SignalFailure, evolution.FailureReasonInferenceError, true)
		return domain.TaskRun{}, fmt.Errorf("generate inference: %w", err)
	}
	st := loopState{
		started:          started,
		basePrompt:       basePrompt,
		round:            0,
		resp:             resp,
		promptTokens:     resp.PromptTokens,
		completionTokens: resp.CompletionTokens,
		cachedTokens:     resp.CachedTokens,
		totalTokens:      resp.TotalTokens,
	}
	return r.runToolLoop(ctx, requestID, agent, task, st)
}
```

Add the `loopState` type (near the top of `runtime.go`, after `Runtime` struct):

```go
// loopState is the mutable state threaded through the tool-execution loop.
// runToolLoop advances it; a suspend serialises the relevant fields to a
// checkpoint and a resume rebuilds it from one.
type loopState struct {
	started          time.Time
	basePrompt       string
	round            int
	toolCtx          []toolEntry
	resp             port.InferenceResponse
	promptTokens     int
	completionTokens int
	cachedTokens     int
	totalTokens      int
}
```

Add `runToolLoop` (this is the old loop body, with the gate check and checkpoint delete added):

```go
// runToolLoop advances the tool-execution loop from st until the model stops
// requesting tools (or the round budget is exhausted), then finalises the run.
// Before executing each round's tool calls it consults the ToolGate: if the gate
// says suspend, it writes a checkpoint and returns ErrSuspended, releasing the
// goroutine. A successfully completed run deletes any checkpoint.
func (r *Runtime) runToolLoop(ctx context.Context, requestID string, agent domain.Agent, task domain.Task, st loopState) (domain.TaskRun, error) {
	for st.round < r.maxToolRounds && len(st.resp.ToolCalls) > 0 {
		suspend, err := r.checkSuspend(ctx, task, st)
		if err != nil {
			return domain.TaskRun{}, err
		}
		if suspend {
			return domain.TaskRun{}, ErrSuspended
		}
		results, err := r.executeToolCalls(ctx, agent, task, st.resp.ToolCalls)
		if err != nil {
			_ = r.publishLearning(ctx, agent, task, evolution.SignalFailure, evolution.FailureReasonToolError, true)
			return domain.TaskRun{}, fmt.Errorf("execute model tool calls: %w", err)
		}
		st.toolCtx = mergeToolResults(st.toolCtx, st.resp.ToolCalls, results, r.maxToolResultChars)
		prompt := boundPrompt(st.basePrompt+renderToolEntries(st.toolCtx), r.maxPromptChars)
		st.resp, err = r.generate(ctx, requestID, prompt, task.Images, stablePrefixRunes(prompt, st.basePrompt))
		if err != nil {
			_ = r.publishLearning(ctx, agent, task, evolution.SignalFailure, evolution.FailureReasonInferenceError, true)
			return domain.TaskRun{}, fmt.Errorf("generate inference after tools: %w", err)
		}
		st.promptTokens += st.resp.PromptTokens
		st.completionTokens += st.resp.CompletionTokens
		st.cachedTokens += st.resp.CachedTokens
		st.totalTokens += st.resp.TotalTokens
		st.round++
	}
	if len(st.resp.ToolCalls) > 0 {
		prompt := boundPrompt(st.basePrompt+renderToolEntries(st.toolCtx), r.maxPromptChars)
		finalPrompt := prompt + "\n\n[系统] 工具调用已达上限。请勿再调用、规划或描述任何工具调用，直接基于以上已获取的信息，用自然语言给出对用户问题的最终回答。"
		final, err := r.generateNoTools(ctx, requestID, finalPrompt, task.Images, stablePrefixRunes(finalPrompt, st.basePrompt))
		if err != nil {
			_ = r.publishLearning(ctx, agent, task, evolution.SignalFailure, evolution.FailureReasonInferenceError, true)
			return domain.TaskRun{}, fmt.Errorf("generate final answer after tool budget exhausted: %w", err)
		}
		st.promptTokens += final.PromptTokens
		st.completionTokens += final.CompletionTokens
		st.cachedTokens += final.CachedTokens
		st.totalTokens += final.TotalTokens
		st.resp = final
	}
	return r.finishRun(ctx, requestID, agent, task, st)
}

// checkSuspend consults the ToolGate for the current round's pending calls and,
// when the gate says pause, persists a checkpoint so the run can resume later.
// It returns true only after the checkpoint is safely on disk (fail-loud on
// write error — never suspend with lost state). Nil gate or nil store → false.
func (r *Runtime) checkSuspend(ctx context.Context, task domain.Task, st loopState) (bool, error) {
	if r.toolGate == nil || r.checkpoints == nil {
		return false, nil
	}
	suspend, err := r.toolGate.ShouldSuspend(ctx, task, st.resp.ToolCalls)
	if err != nil {
		return false, fmt.Errorf("tool gate decision for task %s: %w", task.ID, err)
	}
	if !suspend {
		return false, nil
	}
	cp := sessionstate.Checkpoint{
		SchemaVersion:    sessionstate.CheckpointSchemaVersion,
		TaskID:           task.ID,
		AgentID:          agent0ID(task),
		SessionKey:       sessionKeyForTask(task),
		BasePrompt:       st.basePrompt,
		Round:            st.round,
		ToolEntries:      snapshotToolEntries(st.toolCtx),
		PendingCalls:     st.resp.ToolCalls,
		PromptTokens:     st.promptTokens,
		CompletionTokens: st.completionTokens,
		CachedTokens:     st.cachedTokens,
		TotalTokens:      st.totalTokens,
		Images:           task.Images,
		CreatedAt:        time.Now(),
	}
	if err := r.checkpoints.Save(cp); err != nil {
		return false, fmt.Errorf("save checkpoint for task %s: %w", task.ID, err)
	}
	return true, nil
}
```

Replace `agent0ID(task)` above with `task.AgentID` (the task carries its agent id; no helper needed) — i.e. write `AgentID: task.AgentID,`. (This note prevents an undefined-identifier: use `task.AgentID` directly.)

Add `finishRun` (the old tail from the `inference_completed` event through `return run, nil`, now reading from `st`):

```go
// finishRun emits completion events/audit, deletes any checkpoint (the task is
// done, not suspended), and returns the assembled TaskRun.
func (r *Runtime) finishRun(ctx context.Context, requestID string, agent domain.Agent, task domain.Task, st loopState) (domain.TaskRun, error) {
	if err := r.events.Publish(ctx, domain.RuntimeEvent{
		Type:      "inference_completed",
		TaskID:    task.ID,
		Message:   "model inference completed",
		CreatedAt: time.Now(),
	}); err != nil {
		return domain.TaskRun{}, fmt.Errorf("publish inference completed event: %w", err)
	}
	if err := r.audit.Append(ctx, domain.AuditEvent{
		ID:          task.ID + ":model-audit-1",
		RequestID:   requestID,
		SubjectType: "model",
		SubjectID:   task.ID,
		Action:      "model_inference_completed",
		Hash:        "memory",
		CreatedAt:   time.Now(),
	}); err != nil {
		return domain.TaskRun{}, fmt.Errorf("append model audit event: %w", err)
	}
	ended := time.Now()
	run := domain.TaskRun{
		ID:               task.ID + ":run-1",
		TaskID:           task.ID,
		AgentID:          agent.ID,
		StartedAt:        st.started,
		EndedAt:          ended,
		Result:           st.resp.Text,
		ReasoningSummary: st.resp.ReasoningSummary,
		PromptTokens:     st.promptTokens,
		CompletionTokens: st.completionTokens,
		CachedTokens:     st.cachedTokens,
		TotalTokens:      st.totalTokens,
	}
	if err := r.audit.Append(ctx, domain.AuditEvent{
		ID:          task.ID + ":audit-1",
		RequestID:   requestID,
		SubjectType: "task",
		SubjectID:   task.ID,
		Action:      "task_completed",
		Hash:        "memory",
		CreatedAt:   time.Now(),
	}); err != nil {
		return domain.TaskRun{}, fmt.Errorf("append audit event: %w", err)
	}
	if err := r.events.Publish(ctx, domain.RuntimeEvent{
		Type:             "task_completed",
		TaskID:           task.ID,
		Message:          st.resp.Text,
		PromptTokens:     st.promptTokens,
		CompletionTokens: st.completionTokens,
		CachedTokens:     st.cachedTokens,
		TotalTokens:      st.totalTokens,
		ElapsedMs:        ended.Sub(st.started).Milliseconds(),
		CreatedAt:        time.Now(),
	}); err != nil {
		return domain.TaskRun{}, fmt.Errorf("publish task completed event: %w", err)
	}
	if r.checkpoints != nil {
		if err := r.checkpoints.Delete(sessionKeyForTask(task)); err != nil {
			return domain.TaskRun{}, fmt.Errorf("delete checkpoint after completion for task %s: %w", task.ID, err)
		}
	}
	if err := r.publishLearning(ctx, agent, task, evolution.SignalSuccess, "", false); err != nil {
		return domain.TaskRun{}, fmt.Errorf("publish learning success event: %w", err)
	}
	return run, nil
}
```

Ensure `port` is imported in `runtime.go` (it already is — `port.InferenceResponse` is used in signatures).

- [ ] **Step 4: Run new tests + full runtime suite**

Run: `go test ./internal/runtime/`
Expected: PASS — suspend/resume tests green AND every pre-existing runtime test still green (they use nil gate/store, so `checkSuspend` returns false and behaviour is identical). Pay special attention to the existing tool-budget-exhaustion and dedup tests.

- [ ] **Step 5: Verify build + vet + race**

Run (Windows): `go build ./... && go vet ./...`
Run (WSL): the `-race` command.
Expected: green.

- [ ] **Step 6: Commit**

```bash
git add internal/runtime/runtime.go internal/runtime/checkpoint_test.go
git commit -m "feat(runtime): resumable tool loop — suspend writes checkpoint, entry auto-resumes"
```

---

## Task 5: Coordinator maps `ErrSuspended` → `TaskSuspended`

**Files:**
- Modify: `internal/runtime/coordinator.go` (`runAssigned` RunTask error handling ~195-199)
- Create: `internal/runtime/coordinator_suspend_test.go`

**Interfaces:**
- Consumes: `ErrSuspended` (Task 3), existing `Coordinator`, `task.Scheduler`.
- The scheduler transition table already permits `Running→Suspended` (`scheduler.go:114-117`) and `Suspended→Running` (`scheduler.go:118-119`) — no change there.

- [ ] **Step 1: Write the failing coordinator test**

Create `internal/runtime/coordinator_suspend_test.go`:

```go
package runtime

import (
	"context"
	"testing"

	"github.com/stardust/legion-agent/internal/domain"
	"github.com/stardust/legion-agent/internal/task"
)

// suspendingRunner always returns ErrSuspended, modelling a runtime that
// checkpointed and paused.
type suspendingRunner struct{}

func (suspendingRunner) RunTask(context.Context, domain.Agent, domain.Task) (domain.TaskRun, error) {
	return domain.TaskRun{}, ErrSuspended
}

func TestCoordinatorSuspendedRunLandsSuspendedNotFailed(t *testing.T) {
	sched := task.NewScheduler()
	ctx := context.Background()
	if err := sched.Add(ctx, domain.Task{ID: "t-suspend", AgentID: "default-agent", Status: domain.TaskPending, Input: "x"}); err != nil {
		t.Fatalf("add: %v", err)
	}
	coord := newTestCoordinatorWithRunner(t, sched, suspendingRunner{})

	if _, _, err := coord.Heartbeat(ctx); err != nil {
		t.Fatalf("heartbeat: %v", err)
	}
	coord.Wait()

	got, ok, err := sched.Get(ctx, "t-suspend")
	if err != nil {
		t.Fatalf("get: %v", err)
	}
	if !ok {
		t.Fatal("task missing")
	}
	if got.Status != domain.TaskSuspended {
		t.Errorf("status = %s, want %s", got.Status, domain.TaskSuspended)
	}
}
```

> `newTestCoordinatorWithRunner` may not exist. If the existing `newTestCoordinator(t, sched, workers)` helper (used in `coordinator_concurrency_test.go`) hard-codes its runner, add a sibling helper in the SAME test file that already defines `newTestCoordinator` (search for it — likely `coordinator_test.go`) that accepts a `TaskRunner`. Keep the rest of the CoordinatorConfig (Locks, Reviewer, Evaluator, Approvals, Audit, Events) identical to the existing helper so only the runner differs.

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./internal/runtime/ -run TestCoordinatorSuspendedRun`
Expected: FAIL — task lands in `TaskFailed` (current code transitions to Failed on any RunTask error), or a compile error for the missing helper (add it first, then observe the Failed-vs-Suspended failure).

- [ ] **Step 3: Handle `ErrSuspended` in `runAssigned`**

In `internal/runtime/coordinator.go`, replace the RunTask error block (~195-199):

```go
	run, err := runner.RunTask(ctx, runnerAgent, taskToRun)
	if err != nil {
		if errors.Is(err, ErrSuspended) {
			// The runtime checkpointed and paused (e.g. awaiting approval). Land
			// the task in Suspended — NOT Failed — and release the goroutine. A
			// later decision (or restart recovery) transitions it back to Running
			// and the runtime auto-resumes from its checkpoint.
			if txErr := c.scheduler.Transition(ctx, taskToRun.ID, domain.TaskSuspended); txErr != nil {
				return domain.Task{}, false, fmt.Errorf("suspend checkpointed task %s: %w", taskToRun.ID, txErr)
			}
			if auErr := c.appendAudit(ctx, taskToRun.ID, "task_suspended"); auErr != nil {
				return domain.Task{}, false, auErr
			}
			return c.currentTask(ctx, taskToRun.ID)
		}
		_ = c.scheduler.Transition(ctx, taskToRun.ID, domain.TaskFailed)
		return domain.Task{}, false, fmt.Errorf("run task: %w", err)
	}
```

Add `"errors"` to `coordinator.go`'s import block if not already present.

- [ ] **Step 4: Run test to verify it passes**

Run: `go test ./internal/runtime/ -run TestCoordinatorSuspendedRun`
Expected: PASS.

- [ ] **Step 5: Run full runtime suite + race**

Run (Windows): `go test ./internal/runtime/ ./internal/task/ && go vet ./...`
Run (WSL): the `-race` command.
Expected: green (concurrency stress test still passes — suspended is already an accepted terminal-ish state there).

- [ ] **Step 6: Commit**

```bash
git add internal/runtime/coordinator.go internal/runtime/coordinator_suspend_test.go internal/runtime/coordinator_test.go
git commit -m "feat(runtime): coordinator lands ErrSuspended as TaskSuspended, not Failed"
```

---

## Task 6: Restart recovery + end-to-end suspend→restart→resume + serve wiring

**Files:**
- Modify: `internal/runtime/coordinator.go` — add `RecoverSuspended`.
- Create: `internal/runtime/resume_e2e_test.go`
- Modify: `internal/cli/command.go` — construct `sessionstate.Store` from `workspace.root`, inject into the default runtime (~1804 `NewRuntime`/`RuntimeConfig` block), call `coordinator.RecoverSuspended` after assembly (~1825), and log the workspace.root warning.

**Interfaces:**
- Consumes: `sessionstate.Store.ListSuspended`, `task.Scheduler.Add`, `domain.Task`.
- Produces: `func (c *Coordinator) RecoverSuspended(ctx context.Context, store *sessionstate.Store) (int, error)` — re-registers each checkpointed task into the scheduler in `TaskSuspended` state (so it survives a restart and is visible/decidable); returns the count recovered. It does NOT auto-resume (resume is triggered by a Suspended→Running decision, which arrives from M2's approval layer or a test).

- [ ] **Step 1: Write the failing end-to-end resume-across-restart test**

Create `internal/runtime/resume_e2e_test.go`:

```go
package runtime

import (
	"context"
	"errors"
	"testing"

	"github.com/stardust/legion-agent/internal/adapter"
	"github.com/stardust/legion-agent/internal/domain"
	"github.com/stardust/legion-agent/internal/sessionstate"
)

// TestSuspendRestartResumeEndToEnd proves the core M1b guarantee: a task that
// suspends (checkpoint on disk) can be resumed by a FRESH runtime built over the
// same store directory (simulating a process restart) and finishes with the
// correct result.
func TestSuspendRestartResumeEndToEnd(t *testing.T) {
	dir := t.TempDir()
	agent := domain.Agent{ID: "agent-1"}
	task := domain.Task{ID: "task-1", SessionID: "sess-1", AgentID: "agent-1", Status: domain.TaskRunning, Input: "go"}
	ctx := context.Background()

	// --- process 1: run until it suspends ---
	store1 := sessionstate.NewStore(dir)
	runner1 := NewRuntime(Config{
		Maas:        &scriptedMaas{},
		Audit:       adapter.NewMemoryAuditLog(),
		Events:      adapter.NewMemoryEventBus(),
		Tools:       echoRegistry(t),
		Checkpoints: store1,
		ToolGate:    &gateOnce{}, // suspends once
	})
	if _, err := runner1.RunTask(ctx, agent, task); !errors.Is(err, ErrSuspended) {
		t.Fatalf("process1 run err = %v, want ErrSuspended", err)
	}

	// --- simulate restart: brand-new store + runtime over the same dir ---
	store2 := sessionstate.NewStore(dir)
	suspended, err := store2.ListSuspended()
	if err != nil {
		t.Fatalf("ListSuspended: %v", err)
	}
	if len(suspended) != 1 || suspended[0].TaskID != "task-1" {
		t.Fatalf("recovered = %#v, want one checkpoint for task-1", suspended)
	}

	// Fresh runtime; its gate never suspends (decision has "arrived"), so the
	// resumed loop runs the pending echo call and completes.
	runner2 := NewRuntime(Config{
		Maas:        &scriptedMaas{},
		Audit:       adapter.NewMemoryAuditLog(),
		Events:      adapter.NewMemoryEventBus(),
		Tools:       echoRegistry(t),
		Checkpoints: store2,
		ToolGate:    nil,
	})
	run, err := runner2.RunTask(ctx, agent, task)
	if err != nil {
		t.Fatalf("process2 resume err = %v, want nil", err)
	}
	if run.Result != "final answer" {
		t.Errorf("resumed result = %q, want %q", run.Result, "final answer")
	}
	if _, ok, _ := store2.Load("sess-1"); ok {
		t.Error("checkpoint not cleaned after resume completion")
	}
}

func TestRecoverSuspendedReRegistersTasks(t *testing.T) {
	dir := t.TempDir()
	store := sessionstate.NewStore(dir)
	if err := store.Save(sessionstate.Checkpoint{
		SchemaVersion: sessionstate.CheckpointSchemaVersion,
		TaskID:        "task-9",
		AgentID:       "default-agent",
		SessionKey:    "sess-9",
	}); err != nil {
		t.Fatalf("seed checkpoint: %v", err)
	}
	sched := taskScheduler(t) // helper returning *task.Scheduler
	coord := newTestCoordinator(t, sched, 4)
	ctx := context.Background()

	n, err := coord.RecoverSuspended(ctx, store)
	if err != nil {
		t.Fatalf("RecoverSuspended: %v", err)
	}
	if n != 1 {
		t.Fatalf("recovered count = %d, want 1", n)
	}
	got, ok, err := sched.Get(ctx, "task-9")
	if err != nil || !ok {
		t.Fatalf("get task-9: ok=%v err=%v", ok, err)
	}
	if got.Status != domain.TaskSuspended {
		t.Errorf("recovered task status = %s, want %s", got.Status, domain.TaskSuspended)
	}
}
```

> `taskScheduler(t)` is shorthand — use `task.NewScheduler()` directly (import `internal/task`) if no such helper exists; drop the helper reference.

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./internal/runtime/ -run 'TestSuspendRestartResume|TestRecoverSuspended'`
Expected: FAIL — `TestSuspendRestartResume...` may already pass (auto-resume from Task 4), but `TestRecoverSuspended...` fails with `undefined: (*Coordinator).RecoverSuspended`.

- [ ] **Step 3: Implement `RecoverSuspended`**

In `internal/runtime/coordinator.go`, add:

```go
// RecoverSuspended re-registers every task that has a persisted checkpoint into
// the scheduler in TaskSuspended state, so suspends survive a process restart and
// remain visible/decidable. It does not resume them — resume is driven by a
// Suspended→Running decision. Returns the number of tasks recovered. A task that
// is already present in the scheduler is skipped (idempotent re-scan).
func (c *Coordinator) RecoverSuspended(ctx context.Context, store *sessionstate.Store) (int, error) {
	if store == nil {
		return 0, nil
	}
	checkpoints, err := store.ListSuspended()
	if err != nil {
		return 0, fmt.Errorf("list suspended checkpoints: %w", err)
	}
	recovered := 0
	for _, cp := range checkpoints {
		if _, ok, err := c.scheduler.Get(ctx, cp.TaskID); err != nil {
			return recovered, fmt.Errorf("check task %s presence: %w", cp.TaskID, err)
		} else if ok {
			continue
		}
		if err := c.scheduler.Add(ctx, domain.Task{
			ID:        cp.TaskID,
			AgentID:   cp.AgentID,
			SessionID: cp.SessionKey,
			Status:    domain.TaskSuspended,
		}); err != nil {
			return recovered, fmt.Errorf("re-register suspended task %s: %w", cp.TaskID, err)
		}
		if err := c.appendAudit(ctx, cp.TaskID, "task_recovered_suspended"); err != nil {
			return recovered, err
		}
		recovered++
	}
	return recovered, nil
}
```

Add the `sessionstate` import to `coordinator.go`.

> Note the scheduler `Add` sets status directly from the `domain.Task` passed (see `scheduler.go:28-40` — it stores the task as-is). Confirm `Add` does not force `TaskPending`; the read at Task-1 time showed it stores `task` verbatim, so a `TaskSuspended` task stays suspended and is NOT picked by `Next` (which only advances `TaskPending`). Good — recovered tasks won't be auto-run until a decision flips them to Running (via a future `Transition` that Add→Suspended→Running path allows... note: the transition table has no `Suspended`-from-`Add` issue since Add bypasses transitions). If `Add` is later changed to reject non-pending statuses, revisit this.

- [ ] **Step 4: Run tests to verify they pass**

Run: `go test ./internal/runtime/`
Expected: PASS — both new tests plus all prior.

- [ ] **Step 5: Wire the store into serve assembly**

In `internal/cli/command.go`, in the `serve` assembly (around the default-runtime `NewRuntime` at ~1804 and `NewCoordinator` at ~1807):

a) After `cfg` is available and before building the default runtime, resolve the workspace root and build the store:

```go
	workspaceRoot, rootWarning := sessionstate.ResolveWorkspaceRoot(cfg.Workspace.Root)
	if rootWarning != "" {
		// Fail-loud: a misconfigured workspace.root is surfaced, not swallowed.
		log.Printf("warn: %s", rootWarning) // use the project logger in scope here
	}
	checkpointStore := sessionstate.NewStore(workspaceRoot)
```

> Use whatever structured logger the surrounding `serve` code already uses (search nearby for `log.`/`slog`/a `logger` variable) instead of a bare `log.Printf` — match the file's existing logging idiom per CLAUDE.md (no bare `fmt.Println`/`log.Printf` for warnings if a structured logger exists).

b) Add `Checkpoints: checkpointStore` to the default-runtime `runtime.Config` (the `NewRuntime(runtime.Config{...})` literal at ~1786-1805). Leave `ToolGate` unset (nil) — Auto behaviour until M2.

c) After `coordinator := agentruntime.NewCoordinator(...)` (~1807-1825), recover suspended tasks:

```go
	if _, err := coordinator.RecoverSuspended(ctx, checkpointStore); err != nil {
		closeStore()
		return ServeResult{}, fmt.Errorf("recover suspended tasks: %w", err)
	}
```

d) Add the import `"github.com/stardust/legion-agent/internal/sessionstate"` to `command.go`.

> The per-agent runtimes created by `AgentRuntimeResolver` do NOT get the store in M1b (only the default runtime). That is acceptable: M1b's guarantee is proven by the default path + unit/e2e tests. Wiring the resolver's runtimes is folded into M2/M3 when per-session mode/working_dir plumb through the resolver. Leave a `// TODO(M2): inject checkpoint store into resolver-built runtimes` comment at the resolver construction site (~1730).

- [ ] **Step 6: Build, vet, full test suite, race**

Run (Windows): `go build ./... && go vet ./... && go test ./...`
Run (WSL): the `-race` command.
Expected: all green. If `command_test.go` constructs `serve` and asserts assembly, ensure it still passes (the store is additive; recovery over an empty/absent dir returns 0, nil).

- [ ] **Step 7: gofmt touched files**

Run: `gofmt -l internal/sessionstate/ internal/runtime/ internal/config/config.go internal/cli/command.go`
Expected: empty output. If any file is listed, `gofmt -w` it (only these files — do not touch unrelated pre-existing debt).

- [ ] **Step 8: Commit**

```bash
git add internal/runtime/coordinator.go internal/runtime/resume_e2e_test.go internal/cli/command.go
git commit -m "feat(serve): restart recovery for suspended tasks + wire checkpoint store"
```

---

## Self-Review (completed by plan author)

**1. Spec coverage** (against design §4.0 / §4.1b + handoff §3 M1b scope):
- Session-dir resolver (working_dir→`.stardust`, else workspace.root; `~` expand; fallback+warn) → Task 1. (working_dir branch is forward-declared via `SessionDir(base,...)`; base=working_dir/.stardust wiring is M3 — M1b callers pass workspaceRoot, matching "no working_dir yet" reality.)
- `workspace.root` config field + default + fallback + warn → Task 1.
- Checkpoint (`task-state.json`): toolCtx/round/pendingCall + tokens serialised, atomic write, fail-loud corrupt/version → Task 2.
- Suspend = write checkpoint + `TaskSuspended` + release goroutine → Tasks 4 (checkpoint+ErrSuspended) + 5 (Suspended transition, goroutine already released by Heartbeat's per-task goroutine returning).
- Resume (incl. after restart) from `task-state.json` → Task 4 (auto-resume on entry) + Task 6 (e2e across fresh runtime/store).
- Restart recovery: scan session dirs, load checkpoints, re-register → Task 6 `RecoverSuspended` + serve wiring.
- Acceptance: `-race` green (every task), suspend→restart→resume e2e (Task 6). ✓
- Deliberately deferred to M2 (documented, not gaps): the ToolGate *implementation* backed by approval + tool `Sensitive` bit; per-call (vs per-round) gate granularity at `dispatchToolCall`; approval-JSON persistence; SSE. M1b ships the `ToolGate` seam + checkpoint mechanism only, per handoff ("M2 才接完整").

**2. Placeholder scan:** No "TBD"/"add error handling"/"similar to Task N". Every code step shows real Go. Two flagged verification notes (tool.Registry API in Task 4 Step 1; `newTestCoordinator` helper shape in Task 5 Step 1) direct the implementer to confirm exact existing signatures — these are real-codebase reconciliations, not placeholders; the asserted behaviour is fully specified.

**3. Type consistency:** `Checkpoint`/`ToolEntrySnapshot`/`Store` fields identical across Tasks 2/3/4/6. `sessionKeyForTask`, `snapshotToolEntries`/`restoreToolEntries`, `loopState`, `runToolLoop`/`checkSuspend`/`finishRun`, `RecoverSuspended` names consistent across definition and use. `ErrSuspended` defined Task 3, used Tasks 4/5/6. Corrected one inline slip (`agent0ID(task)` → `task.AgentID`) in Task 4 Step 3.

---

## Execution Handoff

Recommended: **Subagent-Driven** (fresh subagent per task + two-stage review), per handoff §5 note — implementers/reviewers must write reports to files (e.g. `.sdd-m1b/task-N-report.md`) because OMC/caveman hooks eat subagent final messages; verify by reading those files and re-running the WSL `-race` recipe yourself.
