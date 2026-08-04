# Agent 内置浏览器 Phase 3（会话持久化 + TTL 回收 + 重启恢复）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让浏览器会话的登录态跨进程重启存活——storageState（cookies）落 SQLite `agent.db`，空闲会话按 TTL 回收物理 Context（登录态先落盘），再次访问或重启后从盘重建 Context 并恢复登录态。

**Architecture:** 给 `internal/browser` 加一个可选的持久化端口 `BrowserSessionStore`（`nil` = 纯内存，Phase 1/2 行为不变）。`SessionStore` 写穿到该端口（**字段级写穿，先落盘后改内存**——见 [[legion-task-state-persistence]]）。Runtime 在 Close/TTL 回收时用 go-rod `GetCookies` 抓 storageState 落盘、`ReleaseContext` 后置 `Context=nil`；动作命中 `Context==nil` 的会话时重建（acquire → `SetCookies` → navigate 到持久化的 active_url）。持久化实现是 `internal/storage` 的 `SQLiteRepository` 加一张 `browser_sessions` 表（迁移版本 +1）+ 一组 CRUD 方法，经 cli 接线注入 Runtime。

**Tech Stack:** Go 1.26、go-rod（`Browser.GetCookies`/`SetCookies`）、现有 `internal/storage/sqlite.go`（`database/sql` + `modernc.org/sqlite`，schema_migrations 版本化）。

**范围边界（本 plan 不做）:** localStorage/origins 快照（只做 cookies；localStorage 需 per-origin CDP DOMStorage，列为债务）；下载缓存持久化（下载工具是后续 Phase）；多 tab 状态（只持久化活跃 tab 的 url）；跨机/多租户存储（Phase 13 分叉）。

**关联文档:** spec `docs/superpowers/specs/2026-08-04-agent-browser-design.md`（§3.3/§4/§7.3）；Phase 1/2 plan 同目录；持久化教训 [[legion-task-state-persistence]]。

---

## 锁定的设计决策

| 决策 | 选择 | 理由 |
|---|---|---|
| storageState 内容 | **仅 cookies**（`GetCookies`/`SetCookies`） | go-rod 原生、够覆盖多数登录态；localStorage 列债务 |
| 落盘位置 | SQLite `agent.db` 新表 `browser_sessions` | Phase 1 已定复用 agent.db，不引 bbolt |
| 写穿粒度 | **字段级**：元数据(last_used_at/active_url)动作时写穿；storageState 仅在回收/Close 抓+落 | [[legion-task-state-persistence]]：全行 UPSERT 会用旧值覆盖并发写 |
| 落盘/内存顺序 | **先落盘再改内存**（回收：抓 cookies→落盘→ReleaseContext→Context=nil） | 落盘失败不能已经丢了 Context |
| TTL 回收 | 后台 reaper goroutine（间隔可配，默认 60s）；回收 `LastUsedAt+TTL<now` 的会话 | spec §7.3 |
| 未知会话语义 | 内存无 **且** DB 无 → `CONTEXT_EVICTED`；内存无但 DB 有 → 重建 | 修订 Phase 1 I4：evicted≠unknown |
| 端口可选 | `RuntimeConfig.Store` 为 nil → 纯内存（Phase 1/2 不变） | 默认关，不破坏现有测试/无 DB 场景 |

---

## 文件结构

| 文件 | 职责 | 状态 |
|---|---|---|
| `internal/storage/sqlite.go` | `browser_sessions` 建表（append `schemaStatements`）+ bump `CurrentSchemaVersion` + CRUD 方法 | 修改（Task 1） |
| `internal/storage/browser_sessions.go` | `BrowserSessionRecord` 类型 + Save/Get/List/Delete 实现（若 sqlite.go 过大则拆此文件） | 创建（Task 1） |
| `internal/browser/store.go` | `BrowserSessionStore` 端口接口 + `SessionRecord` + cookie 序列化类型 | 创建（Task 2） |
| `internal/browser/session.go` | `SessionStore` 加可选 `persist BrowserSessionStore`，写穿 Create/touch/Delete | 修改（Task 2） |
| `internal/browser/runtime.go` | capture/restore storageState；重建；启动加载；reaper 触发点 | 修改（Task 3/4/5） |
| `internal/browser/reaper.go` | TTL 空闲回收（`reapIdle(now)` + 后台 goroutine） | 创建（Task 4） |
| `internal/cli/command.go`（+ `internal/app/app.go`） | SQLite-backed `BrowserSessionStore` 适配器 + 注入 `NewRuntime` + config | 修改（Task 6） |
| `internal/config/config.go` | `Browser` 配置加 `SessionTTL`/`ReapInterval` | 修改（Task 6） |

---

## Task 1: SQLite `browser_sessions` 表 + CRUD

**Files:**
- Modify: `internal/storage/sqlite.go`（`schemaStatements` 追加建表；bump `CurrentSchemaVersion`）
- Create: `internal/storage/browser_sessions.go`
- Test: `internal/storage/browser_sessions_test.go`

- [ ] **Step 1: 先读现有约定**

Run（了解 `schemaStatements`、`CurrentSchemaVersion`、`formatTime`/`parseTime`/`boolToInt` helper 的确切名字与位置）:
```
grep -n "schemaStatements\|CurrentSchemaVersion\|func formatTime\|func parseTime\|func boolToInt\|func intToBool" internal/storage/sqlite.go
```
据此 Step 3/4 用对 helper 名。

- [ ] **Step 2: 写失败测试**

Create `internal/storage/browser_sessions_test.go`:

```go
package storage

import (
	"context"
	"path/filepath"
	"testing"
	"time"
)

func openTestRepo(t *testing.T) *SQLiteRepository {
	t.Helper()
	repo, err := OpenSQLite(context.Background(), filepath.Join(t.TempDir(), "test.db"))
	if err != nil {
		t.Fatalf("OpenSQLite: %v", err)
	}
	t.Cleanup(func() { _ = repo.Close() })
	return repo
}

func TestBrowserSessionSaveGet(t *testing.T) {
	repo := openTestRepo(t)
	ctx := context.Background()
	now := time.Now().UTC().Truncate(time.Second)
	rec := BrowserSessionRecord{
		ID: "sess-1", TaskID: "t1", ActiveURL: "https://example.com",
		StorageState: `[{"name":"sid","value":"abc"}]`,
		CreatedAt: now, LastUsedAt: now, Evicted: true,
	}
	if err := repo.SaveBrowserSession(ctx, rec); err != nil {
		t.Fatalf("SaveBrowserSession: %v", err)
	}
	got, ok, err := repo.GetBrowserSession(ctx, "sess-1")
	if err != nil || !ok {
		t.Fatalf("GetBrowserSession ok=%v err=%v", ok, err)
	}
	if got.TaskID != "t1" || got.ActiveURL != "https://example.com" || got.StorageState != rec.StorageState || !got.Evicted {
		t.Fatalf("roundtrip mismatch: %+v", got)
	}
}

// 字段级写穿：更新 last_used_at/active_url 不应清空已存的 storage_state。
func TestBrowserSessionTouchPreservesStorageState(t *testing.T) {
	repo := openTestRepo(t)
	ctx := context.Background()
	now := time.Now().UTC().Truncate(time.Second)
	_ = repo.SaveBrowserSession(ctx, BrowserSessionRecord{
		ID: "sess-1", TaskID: "t1", StorageState: `[{"name":"sid"}]`, CreatedAt: now, LastUsedAt: now,
	})
	later := now.Add(time.Minute)
	if err := repo.TouchBrowserSession(ctx, "sess-1", "https://x.com", later); err != nil {
		t.Fatalf("TouchBrowserSession: %v", err)
	}
	got, _, _ := repo.GetBrowserSession(ctx, "sess-1")
	if got.StorageState != `[{"name":"sid"}]` {
		t.Fatalf("touch clobbered storage_state: %q", got.StorageState)
	}
	if got.ActiveURL != "https://x.com" || !got.LastUsedAt.Equal(later) {
		t.Fatalf("touch didn't update url/time: %+v", got)
	}
}

func TestBrowserSessionListAndDelete(t *testing.T) {
	repo := openTestRepo(t)
	ctx := context.Background()
	now := time.Now().UTC().Truncate(time.Second)
	_ = repo.SaveBrowserSession(ctx, BrowserSessionRecord{ID: "a", CreatedAt: now, LastUsedAt: now})
	_ = repo.SaveBrowserSession(ctx, BrowserSessionRecord{ID: "b", CreatedAt: now, LastUsedAt: now})
	list, err := repo.ListBrowserSessions(ctx)
	if err != nil || len(list) != 2 {
		t.Fatalf("list len=%d err=%v", len(list), err)
	}
	if err := repo.DeleteBrowserSession(ctx, "a"); err != nil {
		t.Fatalf("delete: %v", err)
	}
	if _, ok, _ := repo.GetBrowserSession(ctx, "a"); ok {
		t.Fatal("expected a deleted")
	}
}
```

- [ ] **Step 3: 跑，确认失败**

Run: `go test ./internal/storage/ -run TestBrowserSession -v`
Expected: FAIL（`BrowserSessionRecord`/`SaveBrowserSession` 等未定义）。

- [ ] **Step 4: 建表 + 实现**

在 `internal/storage/sqlite.go` 的 `schemaStatements` 列表追加（对齐现有条目风格）：
```sql
CREATE TABLE IF NOT EXISTS browser_sessions (
    id            TEXT PRIMARY KEY,
    task_id       TEXT NOT NULL DEFAULT '',
    active_url    TEXT NOT NULL DEFAULT '',
    storage_state TEXT NOT NULL DEFAULT '',
    created_at    TEXT NOT NULL,
    last_used_at  TEXT NOT NULL,
    evicted       INTEGER NOT NULL DEFAULT 0
);
```
并把 `CurrentSchemaVersion` 常量 +1（新增一张表算一次 schema 升级；建表用 `IF NOT EXISTS` 幂等，无需 applyColumnMigrations）。

Create `internal/storage/browser_sessions.go`:

```go
package storage

import (
	"context"
	"database/sql"
	"fmt"
	"time"
)

// BrowserSessionRecord 是一个内置浏览器会话的持久化快照（登录态 = cookies JSON）。
type BrowserSessionRecord struct {
	ID           string
	TaskID       string
	ActiveURL    string
	StorageState string // cookies 的 JSON 序列化（[]NetworkCookieParam 形状）
	CreatedAt    time.Time
	LastUsedAt   time.Time
	Evicted      bool // 物理 Context 已被回收，下次访问需重建
}

// SaveBrowserSession upsert 整条记录（用于回收/Close 时写入含 storage_state 的完整快照）。
func (r *SQLiteRepository) SaveBrowserSession(ctx context.Context, rec BrowserSessionRecord) error {
	_, err := r.db.ExecContext(ctx, `
		INSERT INTO browser_sessions (id, task_id, active_url, storage_state, created_at, last_used_at, evicted)
		VALUES (?, ?, ?, ?, ?, ?, ?)
		ON CONFLICT(id) DO UPDATE SET
			task_id = excluded.task_id,
			active_url = excluded.active_url,
			storage_state = excluded.storage_state,
			last_used_at = excluded.last_used_at,
			evicted = excluded.evicted
	`, rec.ID, rec.TaskID, rec.ActiveURL, rec.StorageState,
		formatTime(rec.CreatedAt), formatTime(rec.LastUsedAt), boolToInt(rec.Evicted))
	if err != nil {
		return fmt.Errorf("save browser session %q: %w", rec.ID, err)
	}
	return nil
}

// TouchBrowserSession 只更新 active_url 与 last_used_at（字段级写穿，绝不触碰
// storage_state——动作路径没有新 cookies 快照，全行 UPSERT 会用空串覆盖登录态）。
// 记录不存在时报错（fail-loud：touch 一个未持久化的会话是调用方 bug）。
func (r *SQLiteRepository) TouchBrowserSession(ctx context.Context, id, activeURL string, lastUsed time.Time) error {
	res, err := r.db.ExecContext(ctx, `
		UPDATE browser_sessions SET active_url = ?, last_used_at = ?, evicted = 0 WHERE id = ?
	`, activeURL, formatTime(lastUsed), id)
	if err != nil {
		return fmt.Errorf("touch browser session %q: %w", id, err)
	}
	n, _ := res.RowsAffected()
	if n == 0 {
		return fmt.Errorf("touch browser session %q: no such session", id)
	}
	return nil
}

func (r *SQLiteRepository) GetBrowserSession(ctx context.Context, id string) (BrowserSessionRecord, bool, error) {
	row := r.db.QueryRowContext(ctx, `
		SELECT id, task_id, active_url, storage_state, created_at, last_used_at, evicted
		FROM browser_sessions WHERE id = ?
	`, id)
	rec, err := scanBrowserSession(row)
	if err == sql.ErrNoRows {
		return BrowserSessionRecord{}, false, nil
	}
	if err != nil {
		return BrowserSessionRecord{}, false, fmt.Errorf("get browser session %q: %w", id, err)
	}
	return rec, true, nil
}

func (r *SQLiteRepository) ListBrowserSessions(ctx context.Context) ([]BrowserSessionRecord, error) {
	rows, err := r.db.QueryContext(ctx, `
		SELECT id, task_id, active_url, storage_state, created_at, last_used_at, evicted
		FROM browser_sessions ORDER BY last_used_at DESC
	`)
	if err != nil {
		return nil, fmt.Errorf("list browser sessions: %w", err)
	}
	defer rows.Close()
	var out []BrowserSessionRecord
	for rows.Next() {
		rec, err := scanBrowserSession(rows)
		if err != nil {
			return nil, fmt.Errorf("scan browser session: %w", err)
		}
		out = append(out, rec)
	}
	return out, rows.Err()
}

func (r *SQLiteRepository) DeleteBrowserSession(ctx context.Context, id string) error {
	if _, err := r.db.ExecContext(ctx, `DELETE FROM browser_sessions WHERE id = ?`, id); err != nil {
		return fmt.Errorf("delete browser session %q: %w", id, err)
	}
	return nil
}

type rowScanner interface{ Scan(dest ...any) error }

func scanBrowserSession(s rowScanner) (BrowserSessionRecord, error) {
	var rec BrowserSessionRecord
	var created, lastUsed string
	var evicted int
	if err := s.Scan(&rec.ID, &rec.TaskID, &rec.ActiveURL, &rec.StorageState, &created, &lastUsed, &evicted); err != nil {
		return BrowserSessionRecord{}, err
	}
	rec.CreatedAt = parseTime(created)
	rec.LastUsedAt = parseTime(lastUsed)
	rec.Evicted = evicted != 0
	return rec, nil
}
```

> 注：Step 1 若发现 helper 名不同（如 `formatTime`→别的、`parseTime` 返回 `(time.Time,error)`、`boolToInt`→`btoi`），据实改。`parseTime` 若返回 error，在 `scanBrowserSession` 里处理它而非丢弃。

- [ ] **Step 5: 跑，确认通过**

Run: `go test ./internal/storage/ -run TestBrowserSession -v`
Expected: PASS。也跑 `go test ./internal/storage/ 2>&1 | tail -3` 确认既有 storage 测试（含 schema 版本相关）未被 bump 破坏——若有断言 `CurrentSchemaVersion` 具体值的测试，同步更新它。

- [ ] **Step 6: Commit**

```bash
git add internal/storage/sqlite.go internal/storage/browser_sessions.go internal/storage/browser_sessions_test.go
git commit -m "feat(storage): browser_sessions table with field-level touch"
```
（提交信息附空行 + `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`。）

---

## Task 2: `BrowserSessionStore` 端口 + SessionStore 写穿

**Files:**
- Create: `internal/browser/store.go`
- Modify: `internal/browser/session.go`
- Test: `internal/browser/store_test.go`

- [ ] **Step 1: 写失败测试（假 store，无 Chromium/DB）**

Create `internal/browser/store_test.go`:

```go
package browser

import (
	"testing"
	"time"
)

// fakePersist 记录写穿调用。
type fakePersist struct {
	saved   map[string]SessionRecord
	touched map[string]time.Time
	deleted []string
}

func newFakePersist() *fakePersist {
	return &fakePersist{saved: map[string]SessionRecord{}, touched: map[string]time.Time{}}
}
func (f *fakePersist) Save(rec SessionRecord) error { f.saved[rec.ID] = rec; return nil }
func (f *fakePersist) Touch(id, activeURL string, lastUsed time.Time) error {
	f.touched[id] = lastUsed
	return nil
}
func (f *fakePersist) Get(id string) (SessionRecord, bool, error) {
	r, ok := f.saved[id]
	return r, ok, nil
}
func (f *fakePersist) List() ([]SessionRecord, error) {
	var out []SessionRecord
	for _, r := range f.saved {
		out = append(out, r)
	}
	return out, nil
}
func (f *fakePersist) Delete(id string) error { f.deleted = append(f.deleted, id); return nil }

func TestSessionStoreWritesThroughOnCreate(t *testing.T) {
	p := newFakePersist()
	st := NewSessionStore()
	st.SetPersist(p)
	sess := st.Create("task-1")
	if _, ok := p.saved[sess.ID]; !ok {
		t.Fatalf("Create did not persist session %q", sess.ID)
	}
}

func TestSessionStoreDeleteWritesThrough(t *testing.T) {
	p := newFakePersist()
	st := NewSessionStore()
	st.SetPersist(p)
	sess := st.Create("task-1")
	st.Delete(sess.ID)
	if len(p.deleted) != 1 || p.deleted[0] != sess.ID {
		t.Fatalf("Delete did not persist removal: %v", p.deleted)
	}
}

// persist 为 nil（默认）时不 panic——Phase 1/2 纯内存路径不变。
func TestSessionStoreNilPersistNoPanic(t *testing.T) {
	st := NewSessionStore()
	sess := st.Create("task-1")
	st.Delete(sess.ID)
	_ = sess
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/browser/ -run TestSessionStore -v`
Expected: FAIL（`SessionRecord`/`SetPersist`/`BrowserSessionStore` 未定义）。

- [ ] **Step 3: 实现端口 + 写穿**

Create `internal/browser/store.go`:

```go
package browser

import "time"

// SessionRecord 是持久化一个浏览器会话所需的字段（与 storage.BrowserSessionRecord 对应，
// 但 browser 包不依赖 storage 包——由 cli 侧适配器桥接，避免层反向依赖）。
type SessionRecord struct {
	ID           string
	TaskID       string
	ActiveURL    string
	StorageState string // cookies JSON
	CreatedAt    time.Time
	LastUsedAt   time.Time
	Evicted      bool
}

// BrowserSessionStore 是可选的持久化端口。nil 表示纯内存。
type BrowserSessionStore interface {
	Save(rec SessionRecord) error                                // 完整快照（含 storage_state）
	Touch(id, activeURL string, lastUsed time.Time) error        // 字段级：只动 url/时间，不碰 storage_state
	Get(id string) (SessionRecord, bool, error)
	List() ([]SessionRecord, error)
	Delete(id string) error
}
```

Edit `internal/browser/session.go`：给 `SessionStore` 加 persist 字段与写穿：
```go
// 在 SessionStore struct 里加：
	persist BrowserSessionStore

// SetPersist 装配可选持久化端口（nil = 纯内存，Phase 1/2 行为不变）。
func (st *SessionStore) SetPersist(p BrowserSessionStore) { st.persist = p }
```
在 `Create` 末尾（返回前）加写穿（**先内存后落盘，落盘失败记 Warn 不致命**——会话已在内存可用，持久化是尽力而为的登录态保险；但记 Warn 以便排查）：
```go
	if st.persist != nil {
		if err := st.persist.Save(SessionRecord{
			ID: sess.ID, TaskID: sess.TaskID,
			CreatedAt: sess.CreatedAt, LastUsedAt: sess.LastUsedAt,
		}); err != nil {
			slog.Warn("persist browser session on create failed", "session", sess.ID, "err", err)
		}
	}
```
在 `Delete` 里删内存后加：
```go
	if st.persist != nil {
		if err := st.persist.Delete(id); err != nil {
			slog.Warn("persist browser session delete failed", "session", id, "err", err)
		}
	}
```
加 `import "log/slog"`（若包内已用别的 logger 约定，Step 前先 `grep -n "slog\|logger" internal/browser/*.go` 对齐；无则用 `slog`）。

> **顺序说明**：Create/Delete 用「先内存后落盘 + Warn」，因为这些是元数据、丢了可从下次动作或重启列表恢复；而 storageState 的落盘（Task 3 回收路径）用「先落盘后置 Context=nil」——登录态丢了不可恢复，顺序相反。两种顺序是有意的，别统一。

- [ ] **Step 4: 跑，确认通过**

Run: `go test ./internal/browser/ -run TestSessionStore -v`
Expected: PASS。也 `go build ./internal/browser/` 干净。

- [ ] **Step 5: Commit**

```bash
git add internal/browser/store.go internal/browser/session.go internal/browser/store_test.go
git commit -m "feat(browser): optional persistence port with write-through session store"
```

---

## Task 3: storageState 抓取/恢复（cookies）

**Files:**
- Modify: `internal/browser/runtime.go`（capture + restore helper；Close 时抓取落盘）
- Test: `internal/browser/storagestate_test.go`（序列化单测）+ `internal/browser/storagestate_integration_test.go`（`//go:build chromium`，真 cookie 往返）

- [ ] **Step 1: 查 go-rod cookie API**

Run:
```
go doc github.com/go-rod/rod Browser.GetCookies
go doc github.com/go-rod/rod Browser.SetCookies
go doc github.com/go-rod/rod/lib/proto NetworkCookie
go doc github.com/go-rod/rod/lib/proto NetworkCookieParam
```
确认 `GetCookies() ([]*proto.NetworkCookie, error)` 与 `SetCookies([]*proto.NetworkCookieParam) error` 的字段（name/value/domain/path/expires/httpOnly/secure/sameSite）。

- [ ] **Step 2: 写序列化单测（无 Chromium）**

Create `internal/browser/storagestate_test.go`:

```go
package browser

import "testing"

func TestStorageStateRoundTripJSON(t *testing.T) {
	cookies := []CookieState{
		{Name: "sid", Value: "abc", Domain: "example.com", Path: "/", Secure: true},
		{Name: "pref", Value: "dark", Domain: "example.com", Path: "/"},
	}
	js, err := marshalStorageState(cookies)
	if err != nil {
		t.Fatalf("marshal: %v", err)
	}
	back, err := unmarshalStorageState(js)
	if err != nil {
		t.Fatalf("unmarshal: %v", err)
	}
	if len(back) != 2 || back[0].Name != "sid" || back[0].Value != "abc" || !back[0].Secure {
		t.Fatalf("roundtrip mismatch: %+v", back)
	}
}

func TestUnmarshalEmptyStorageState(t *testing.T) {
	back, err := unmarshalStorageState("")
	if err != nil || len(back) != 0 {
		t.Fatalf("empty should give nil, got %+v err=%v", back, err)
	}
}
```

- [ ] **Step 3: 跑，确认失败**

Run: `go test ./internal/browser/ -run 'TestStorageState|TestUnmarshal' -v`
Expected: FAIL（`CookieState`/`marshalStorageState`/`unmarshalStorageState` 未定义）。

- [ ] **Step 4: 实现 capture/restore**

Create/extend in `internal/browser/runtime.go`（或新文件 `storagestate.go`，若 runtime.go 已大就拆）:

```go
// CookieState 是一条持久化 cookie（go-rod NetworkCookie 的可序列化子集）。
type CookieState struct {
	Name     string  `json:"name"`
	Value    string  `json:"value"`
	Domain   string  `json:"domain"`
	Path     string  `json:"path"`
	Expires  float64 `json:"expires,omitempty"`
	HTTPOnly bool    `json:"httpOnly,omitempty"`
	Secure   bool    `json:"secure,omitempty"`
	SameSite string  `json:"sameSite,omitempty"`
}

func marshalStorageState(cookies []CookieState) (string, error) {
	if len(cookies) == 0 {
		return "", nil
	}
	b, err := json.Marshal(cookies)
	if err != nil {
		return "", NewBrowserErrorWrap(CodeContextEvicted, "marshal storage state", err)
	}
	return string(b), nil
}

func unmarshalStorageState(js string) ([]CookieState, error) {
	if js == "" {
		return nil, nil
	}
	var cookies []CookieState
	if err := json.Unmarshal([]byte(js), &cookies); err != nil {
		return nil, NewBrowserErrorWrap(CodeContextEvicted, "unmarshal storage state", err)
	}
	return cookies, nil
}

// captureCookies 从会话的 incognito browser 抓当前 cookies。
func (r *Runtime) captureCookies(sess *Session) ([]CookieState, error) {
	if sess == nil || sess.Context == nil || sess.Context.browser == nil {
		return nil, nil
	}
	raw, err := sess.Context.browser.GetCookies()
	if err != nil {
		return nil, NewBrowserErrorWrap(CodeContextEvicted, "get cookies", err)
	}
	out := make([]CookieState, 0, len(raw))
	for _, c := range raw {
		out = append(out, CookieState{
			Name: c.Name, Value: c.Value, Domain: c.Domain, Path: c.Path,
			Expires: float64(c.Expires), HTTPOnly: c.HTTPOnly, Secure: c.Secure,
			SameSite: string(c.SameSite),
		})
	}
	return out, nil
}

// restoreCookies 把持久化 cookies 注入一个新 incognito browser（重建时用）。
func (r *Runtime) restoreCookies(ctxBrowser *rod.Browser, cookies []CookieState) error {
	if ctxBrowser == nil || len(cookies) == 0 {
		return nil
	}
	params := make([]*proto.NetworkCookieParam, 0, len(cookies))
	for _, c := range cookies {
		params = append(params, &proto.NetworkCookieParam{
			Name: c.Name, Value: c.Value, Domain: c.Domain, Path: c.Path,
			Expires: proto.TimeSinceEpoch(c.Expires), HTTPOnly: c.HTTPOnly, Secure: c.Secure,
			SameSite: proto.NetworkCookieSameSite(c.SameSite),
		})
	}
	if err := ctxBrowser.SetCookies(params); err != nil {
		return NewBrowserErrorWrap(CodeContextEvicted, "set cookies", err)
	}
	return nil
}
```
（据 Step 1 的 go doc 调整字段名/类型转换，如 `proto.TimeSinceEpoch`、`NetworkCookieSameSite`；`SetCookies` 的 param 若还需 `URL` 字段，据 domain 拼。若 `json`/`rod`/`proto` 未 import，补齐。）

在 `Close` 里（回收/落盘 storage_state 前的抓取，供 Task 4/5 复用）：Close 单会话时，若有 persist，先 `captureCookies`→`marshalStorageState`→`persist.Save(完整快照 evicted=false→或直接 Delete)`。**本 Task 只提供 capture/restore + 序列化**；把它们接进 Close/回收留给 Task 4/5，避免本 Task 触发 Chromium 依赖过多。

- [ ] **Step 5: 写 chromium 往返集成测试**

Create `internal/browser/storagestate_integration_test.go`:

```go
//go:build chromium

package browser

import (
	"context"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestCaptureRestoreCookiesRoundTrip(t *testing.T) {
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		http.SetCookie(w, &http.Cookie{Name: "sid", Value: "phase3", Path: "/"})
		w.Write([]byte(`<html><body>ok</body></html>`))
	}))
	defer srv.Close()

	rt, err := NewRuntime(RuntimeConfig{Headless: true, AllowPrivateHosts: true, BinPath: systemChromeForTest()})
	if err != nil {
		t.Fatalf("NewRuntime: %v", err)
	}
	defer rt.Close(context.Background(), CloseReq{})

	open, err := rt.Open(context.Background(), OpenReq{URL: srv.URL, TaskID: "t"})
	if err != nil {
		t.Fatalf("Open: %v", err)
	}
	sess, _ := rt.sessions.Get(open.SessionID)
	cookies, err := rt.captureCookies(sess)
	if err != nil {
		t.Fatalf("captureCookies: %v", err)
	}
	var found bool
	for _, c := range cookies {
		if c.Name == "sid" && c.Value == "phase3" {
			found = true
		}
	}
	if !found {
		t.Fatalf("captured cookies missing sid=phase3: %+v", cookies)
	}
}
```
（`systemChromeForTest()` 已在 Phase 2 的 chromium 测试里定义于本包；若不在 build，复制一份小 helper 或复用。）

- [ ] **Step 6: 跑**

Run:
```
go test ./internal/browser/ -run 'TestStorageState|TestUnmarshal' -v
go build ./internal/browser/ && go vet -tags chromium ./internal/browser/
go test -tags chromium ./internal/browser/ -run TestCaptureRestoreCookies -v   # 系统 Chrome
```
Expected: 序列化单测 PASS；tagged-vet 干净；cookie 往返集成 PASS（本机用系统 Chrome）。

- [ ] **Step 7: Commit**

```bash
git add internal/browser/runtime.go internal/browser/storagestate_test.go internal/browser/storagestate_integration_test.go
git commit -m "feat(browser): capture/restore session cookies (storage state)"
```

---

## Task 4: TTL 空闲回收（reaper）

**Files:**
- Create: `internal/browser/reaper.go`
- Modify: `internal/browser/runtime.go`（RuntimeConfig 加 TTL/ReapInterval；启动 reaper；`reapIdle` 逻辑）
- Test: `internal/browser/reaper_test.go`（用注入的 now，无 goroutine 定时）

- [ ] **Step 1: 写失败测试（可注入 now，无 Chromium）**

Create `internal/browser/reaper_test.go`:

```go
package browser

import (
	"testing"
	"time"
)

// reapIdle 应回收 LastUsedAt+TTL < now 的、且有物理 Context 的会话；
// 无 Context（已 evicted）或未过期的跳过。此测试不建真 Context——用 nil Context 验证
// 「选择哪些会话」的判定，回收动作里对 nil Context 安全跳过物理释放。
func TestReapIdleSelectsExpired(t *testing.T) {
	rt := &Runtime{sessions: NewSessionStore(), hubs: newHubRegistry(), cfg: RuntimeConfig{SessionTTL: 10 * time.Minute}}
	now := time.Unix(1_000_000, 0)

	fresh := rt.sessions.Create("t")
	fresh.LastUsedAt = now.Add(-time.Minute) // 未过期
	old := rt.sessions.Create("t")
	old.LastUsedAt = now.Add(-time.Hour) // 过期

	reaped := rt.reapIdle(now)

	if len(reaped) != 1 || reaped[0] != old.ID {
		t.Fatalf("reaped = %v, want [%s]", reaped, old.ID)
	}
	// 过期会话应标记 Context=nil（evicted）
	got, _ := rt.sessions.Get(old.ID)
	if got.Context != nil {
		t.Fatalf("expired session Context should be nil after reap")
	}
}
```

- [ ] **Step 2: 跑，确认失败**

Run: `go test ./internal/browser/ -run TestReapIdle -v`
Expected: FAIL（`RuntimeConfig.SessionTTL`/`reapIdle` 未定义）。

- [ ] **Step 3: 实现 reaper**

在 `RuntimeConfig` 加：`SessionTTL time.Duration` 与 `ReapInterval time.Duration`。

Create `internal/browser/reaper.go`:

```go
package browser

import (
	"context"
	"log/slog"
	"time"
)

// reapIdle 回收所有空闲超过 TTL 的会话：先落盘 storageState，再释放物理 Context、
// 置 Context=nil（会话记录仍在内存与 DB，供之后重建）。返回被回收的会话 id。
// TTL<=0 时不回收。传入 now 便于测试注入。
func (r *Runtime) reapIdle(now time.Time) []string {
	if r.cfg.SessionTTL <= 0 {
		return nil
	}
	var reaped []string
	for _, sess := range r.sessions.Snapshot() { // Snapshot: 复制一份，避免遍历时改 map
		if sess.Context == nil {
			continue // 已 evicted
		}
		if now.Sub(sess.LastUsedAt) < r.cfg.SessionTTL {
			continue
		}
		r.evictSession(sess)
		reaped = append(reaped, sess.ID)
	}
	return reaped
}

// evictSession 落盘登录态 → 释放 Context → 置 nil。先落盘后释放（登录态不可丢）。
func (r *Runtime) evictSession(sess *Session) {
	// 1. 抓 cookies + 落盘（先落盘）
	if store := r.sessions.persist; store != nil {
		cookies, err := r.captureCookies(sess)
		if err != nil {
			slog.Warn("evict: capture cookies failed", "session", sess.ID, "err", err)
		}
		js, _ := marshalStorageState(cookies)
		if err := store.Save(SessionRecord{
			ID: sess.ID, TaskID: sess.TaskID, ActiveURL: r.activeURLOf(sess),
			StorageState: js, CreatedAt: sess.CreatedAt, LastUsedAt: sess.LastUsedAt, Evicted: true,
		}); err != nil {
			slog.Warn("evict: persist storage state failed — keeping Context to avoid losing login", "session", sess.ID, "err", err)
			return // 落盘失败：不释放 Context，避免丢登录态（下次再试）
		}
	}
	// 2. 释放物理 Context（后释放）
	if sess.Context != nil {
		_ = r.mgr.ReleaseContext(sess.Context)
	}
	r.stopScreencast(sess.ID) // Phase 2：会话没了视图也停
	sess.WithLock(func() {
		sess.Context = nil
		sess.ActivePage = nil
	})
}

// startReaper 起后台回收 goroutine，直到 ctx 取消。间隔 ReapInterval（默认 60s）。
func (r *Runtime) startReaper(ctx context.Context) {
	interval := r.cfg.ReapInterval
	if interval <= 0 {
		interval = time.Minute
	}
	if r.cfg.SessionTTL <= 0 {
		return // 未启用 TTL
	}
	go func() {
		ticker := time.NewTicker(interval)
		defer ticker.Stop()
		for {
			select {
			case <-ctx.Done():
				return
			case t := <-ticker.C:
				r.reapIdle(t)
			}
		}
	}()
}
```

在 `internal/browser/session.go` 加 `Snapshot`（复制会话切片）：
```go
// Snapshot 返回当前会话的浅拷贝切片，供并发安全遍历（reaper 用）。
func (st *SessionStore) Snapshot() []*Session {
	st.mu.Lock()
	defer st.mu.Unlock()
	out := make([]*Session, 0, len(st.byID))
	for _, s := range st.byID {
		out = append(out, s)
	}
	return out
}
```
在 `runtime.go` 加 `activeURLOf(sess *Session) string`（读会话当前 URL——从 ActivePage 的 rod page `page.MustInfo().URL` 或维护的字段；若尚未维护 URL，Task 5 会加一个 `sess.ActiveURL` 字段，本 Task 先返回空串占位，Task 5 填实）。**为避免前向依赖**：本 Task 在 `Session` 结构体加 `ActiveURL string` 字段，`Open` 成功导航后写入 `sess.ActiveURL = req.URL`（在 WithLock 内），`activeURLOf` 读它。

- [ ] **Step 4: 跑，确认通过**

Run: `go test ./internal/browser/ -run 'TestReapIdle|TestSessionStore|TestHub' -v`
Expected: PASS。`go build ./...` 干净。

- [ ] **Step 5: Commit**

```bash
git add internal/browser/reaper.go internal/browser/runtime.go internal/browser/session.go internal/browser/reaper_test.go
git commit -m "feat(browser): TTL idle reaper persists login then evicts context"
```

---

## Task 5: 重建 + 启动恢复

**Files:**
- Modify: `internal/browser/runtime.go`（Context==nil 时重建；NewRuntime 起 reaper + 加载持久会话）
- Test: `internal/browser/rebuild_integration_test.go`（`//go:build chromium`）

- [ ] **Step 1: 写 chromium 集成测试（回收后重建保登录 + 模拟重启恢复）**

Create `internal/browser/rebuild_integration_test.go`:

```go
//go:build chromium

package browser

import (
	"context"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"
)

// cookieEcho 服务：/set 下发 cookie；/ 回显收到的 cookie 值。
func cookieEchoServer(t *testing.T) *httptest.Server {
	t.Helper()
	mux := http.NewServeMux()
	mux.HandleFunc("/set", func(w http.ResponseWriter, r *http.Request) {
		http.SetCookie(w, &http.Cookie{Name: "sid", Value: "loginX", Path: "/"})
		w.Write([]byte(`<html><body>set</body></html>`))
	})
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		v := "none"
		if c, err := r.Cookie("sid"); err == nil {
			v = c.Value
		}
		w.Write([]byte(`<html><body>sid=` + v + `</body></html>`))
	})
	s := httptest.NewServer(mux)
	t.Cleanup(s.Close)
	return s
}

func TestRebuildAfterEvictionPreservesLogin(t *testing.T) {
	store := newMemStore() // 见 Step 2 的内存实现（本包测试用）
	srv := cookieEchoServer(t)

	rt, err := NewRuntime(RuntimeConfig{
		Headless: true, AllowPrivateHosts: true, BinPath: systemChromeForTest(),
		SessionTTL: time.Hour, Store: store,
	})
	if err != nil {
		t.Fatalf("NewRuntime: %v", err)
	}
	defer rt.Close(context.Background(), CloseReq{})
	ctx := context.Background()

	open, err := rt.Open(ctx, OpenReq{URL: srv.URL + "/set", TaskID: "t"})
	if err != nil {
		t.Fatalf("Open /set: %v", err)
	}
	sid := open.SessionID

	// 强制回收（模拟 TTL 到）
	sess, _ := rt.sessions.Get(sid)
	rt.evictSession(sess)
	if got, _ := rt.sessions.Get(sid); got.Context != nil {
		t.Fatal("expected evicted (Context nil)")
	}

	// 再次访问该会话 → 应重建 Context、恢复 cookie，导航到根路径回显 loginX
	obs, err := rt.Open(ctx, OpenReq{URL: srv.URL + "/", SessionID: sid})
	if err != nil {
		t.Fatalf("re-Open after evict: %v", err)
	}
	// 读页面文本确认 cookie 生效（用 markdown/read 或 observation 文本）
	read, _ := rt.Read(ctx, ReadReq{SessionID: obs.SessionID})
	_ = read
	// 直接取页面 body 断言
	page := rt.mustActivePageForTest(sid)
	text, _ := page.MustElement("body").Text()
	if !strings.Contains(text, "sid=loginX") {
		t.Fatalf("login not preserved after rebuild; body=%q", text)
	}
}
```

> `newMemStore()` / `mustActivePageForTest` 是本包测试辅助：前者是 `BrowserSessionStore` 的内存实现（map），后者取会话的 `*rod.Page`。在此测试文件内定义它们（`newMemStore` 用一个简单 map+mutex；`mustActivePageForTest` 从 `rt.sessions.Get(id)` 的 `ActivePage.page.(*rod.Page)` 取）。

- [ ] **Step 2: 实现重建 + 启动加载 + 内存 store 辅助**

在 `internal/browser/runtime.go`：

1) `Open` 的会话解析改为支持重建（修订 Phase 1 I4）：
```go
	var sess *Session
	if req.SessionID == "" {
		sess = r.sessions.Create(req.TaskID)
	} else {
		existing, ok := r.sessions.Get(req.SessionID)
		if !ok {
			return OpenObservation{}, NewBrowserError(CodeContextEvicted, "unknown session "+req.SessionID)
		}
		sess = existing
	}
	// 确保有物理 Context（新建 或 重建 evicted 会话）
	if sess.Context == nil {
		if err := r.rebuildContext(sess); err != nil {
			return OpenObservation{}, err
		}
	}
```
2) `rebuildContext`：
```go
// rebuildContext 为一个 Context==nil 的会话获取新 incognito Context 并恢复登录 cookies。
func (r *Runtime) rebuildContext(sess *Session) error {
	c, err := r.mgr.AcquireContext(ContextOpts{})
	if err != nil {
		return err
	}
	// 从持久化恢复 cookies（若有 store 且有快照）
	if store := r.sessions.persist; store != nil {
		if rec, ok, _ := store.Get(sess.ID); ok && rec.StorageState != "" {
			cookies, err := unmarshalStorageState(rec.StorageState)
			if err != nil {
				slog.Warn("rebuild: bad storage state", "session", sess.ID, "err", err)
			} else if err := r.restoreCookies(c.browser, cookies); err != nil {
				slog.Warn("rebuild: restore cookies failed", "session", sess.ID, "err", err)
			}
		}
	}
	sess.WithLock(func() { sess.Context = c })
	return nil
}
```
> 注意：Open 在 `sess.Context==nil` 时先 `rebuildContext`（拿 Context + 灌 cookies），随后既有的导航逻辑 `sess.Context.browser.Page(url)` 会带着 cookies 打开目标页——登录态即恢复。cookies 是 SetCookies 到 browser 级，导航到同域即生效。

3) 启动加载 + reaper：在 `NewRuntime` 末尾，若 `cfg.Store != nil`：`r.sessions.SetPersist(cfg.Store)`，并加载持久会话进内存（Context=nil，懒重建）：
```go
	if cfg.Store != nil {
		r.sessions.SetPersist(cfg.Store)
		if recs, err := cfg.Store.List(); err == nil {
			for _, rec := range recs {
				r.sessions.Adopt(rec) // 见下：把持久记录塞进内存，Context=nil
			}
		} else {
			slog.Warn("load persisted browser sessions failed", "err", err)
		}
	}
	r.startReaper(context.Background()) // reaper 生命周期随进程；Close 时应 cancel（见下）
```
在 `RuntimeConfig` 加 `Store BrowserSessionStore`。给 reaper 一个可取消的 ctx：`Runtime` 加字段 `reaperCancel context.CancelFunc`，`startReaper` 用 `context.WithCancel`，`Close`（全量关闭，`req.SessionID==""` 分支）里调用它。

4) `SessionStore.Adopt`（`session.go`）：
```go
// Adopt 把一条持久化记录纳入内存，Context=nil（懒重建），不回写持久层。
func (st *SessionStore) Adopt(rec SessionRecord) *Session {
	st.mu.Lock()
	defer st.mu.Unlock()
	sess := &Session{
		ID: rec.ID, TaskID: rec.TaskID, ActiveURL: rec.ActiveURL,
		Refs: make(map[string]string), CreatedAt: rec.CreatedAt, LastUsedAt: rec.LastUsedAt,
	}
	st.byID[rec.ID] = sess
	// 维护 seq 单调：若 rec.ID 形如 sess-N，令 st.seq 不小于 N（避免新建撞号）
	return sess
}
```
5) 测试辅助 `mustActivePageForTest`（放 `rebuild_integration_test.go`，非生产码；或加一个不导出的 `activePageRod(sess)` 复用）。

- [ ] **Step 3: 跑集成测试（系统 Chrome）**

Run: `go test -tags chromium ./internal/browser/ -run TestRebuildAfterEviction -v`
Expected: PASS（回收后再访问，cookie loginX 保留，页面回显 `sid=loginX`）。

- [ ] **Step 4: 非集成回归**

Run: `go build ./... && go test ./internal/browser/ ./internal/storage/ ./internal/tool/ ./internal/runtime/ ./internal/server/ 2>&1 | tail -12`
Expected: 全绿（含 drift）。

- [ ] **Step 5: Commit**

```bash
git add internal/browser/runtime.go internal/browser/session.go internal/browser/rebuild_integration_test.go
git commit -m "feat(browser): rebuild evicted context restoring cookies + load persisted sessions on start"
```

---

## Task 6: 接线（SQLite 适配器 + 注入 Runtime + config）

**Files:**
- Create: `internal/cli/browser_store.go`（`storage.SQLiteRepository` → `browser.BrowserSessionStore` 适配器）
- Modify: `internal/cli/command.go`（构造适配器、注入 `browser.NewRuntime`）
- Modify: `internal/app/app.go`（若它也建 Runtime，同样注入；否则跳过）
- Modify: `internal/config/config.go`（`BrowserConfig` 加 `SessionTTL`/`ReapInterval` 字符串或 duration 字段）
- Test: `internal/cli/browser_store_test.go`（适配器桥接正确）

- [ ] **Step 1: 调查现有装配**

Run:
```
grep -n "OpenSQLite\|SQLiteRepository\|sharedBrowser\|browser.NewRuntime\|RuntimeConfig{" internal/cli/command.go internal/app/app.go
grep -n "BrowserConfig\|Browser BrowserConfig" internal/config/config.go
```
找到 SQLite repo 变量、`browser.NewRuntime` 调用点、`BrowserConfig` 定义。

- [ ] **Step 2: 写适配器测试**

Create `internal/cli/browser_store_test.go`:

```go
package cli

import (
	"context"
	"path/filepath"
	"testing"
	"time"

	"github.com/stardust/legion-agent/internal/browser"
	"github.com/stardust/legion-agent/internal/storage"
)

func TestSQLiteBrowserStoreBridge(t *testing.T) {
	repo, err := storage.OpenSQLite(context.Background(), filepath.Join(t.TempDir(), "b.db"))
	if err != nil {
		t.Fatalf("OpenSQLite: %v", err)
	}
	defer repo.Close()
	var store browser.BrowserSessionStore = newSQLiteBrowserStore(repo)

	now := time.Now().UTC().Truncate(time.Second)
	if err := store.Save(browser.SessionRecord{ID: "s1", TaskID: "t", StorageState: `[{"name":"sid"}]`, CreatedAt: now, LastUsedAt: now, Evicted: true}); err != nil {
		t.Fatalf("Save: %v", err)
	}
	got, ok, err := store.Get("s1")
	if err != nil || !ok || got.StorageState != `[{"name":"sid"}]` || !got.Evicted {
		t.Fatalf("Get mismatch ok=%v err=%v rec=%+v", ok, err, got)
	}
	if err := store.Touch("s1", "https://x", now.Add(time.Minute)); err != nil {
		t.Fatalf("Touch: %v", err)
	}
	if again, _, _ := store.Get("s1"); again.StorageState != `[{"name":"sid"}]` {
		t.Fatalf("Touch clobbered storage state")
	}
}
```

- [ ] **Step 3: 实现适配器**

Create `internal/cli/browser_store.go`:

```go
package cli

import (
	"context"
	"time"

	"github.com/stardust/legion-agent/internal/browser"
	"github.com/stardust/legion-agent/internal/storage"
)

// sqliteBrowserStore 把 browser.BrowserSessionStore 端口桥到 storage.SQLiteRepository。
// browser 包不依赖 storage 包；桥接在 cli 层，方向正确。
type sqliteBrowserStore struct {
	repo *storage.SQLiteRepository
	ctx  context.Context
}

func newSQLiteBrowserStore(repo *storage.SQLiteRepository) *sqliteBrowserStore {
	return &sqliteBrowserStore{repo: repo, ctx: context.Background()}
}

func (s *sqliteBrowserStore) Save(rec browser.SessionRecord) error {
	return s.repo.SaveBrowserSession(s.ctx, storage.BrowserSessionRecord{
		ID: rec.ID, TaskID: rec.TaskID, ActiveURL: rec.ActiveURL, StorageState: rec.StorageState,
		CreatedAt: rec.CreatedAt, LastUsedAt: rec.LastUsedAt, Evicted: rec.Evicted,
	})
}
func (s *sqliteBrowserStore) Touch(id, activeURL string, lastUsed time.Time) error {
	return s.repo.TouchBrowserSession(s.ctx, id, activeURL, lastUsed)
}

func (s *sqliteBrowserStore) Get(id string) (browser.SessionRecord, bool, error) {
	rec, ok, err := s.repo.GetBrowserSession(s.ctx, id)
	if err != nil || !ok {
		return browser.SessionRecord{}, ok, err
	}
	return toBrowserRecord(rec), true, nil
}
func (s *sqliteBrowserStore) List() ([]browser.SessionRecord, error) {
	recs, err := s.repo.ListBrowserSessions(s.ctx)
	if err != nil {
		return nil, err
	}
	out := make([]browser.SessionRecord, 0, len(recs))
	for _, r := range recs {
		out = append(out, toBrowserRecord(r))
	}
	return out, nil
}
func (s *sqliteBrowserStore) Delete(id string) error { return s.repo.DeleteBrowserSession(s.ctx, id) }

func toBrowserRecord(r storage.BrowserSessionRecord) browser.SessionRecord {
	return browser.SessionRecord{
		ID: r.ID, TaskID: r.TaskID, ActiveURL: r.ActiveURL, StorageState: r.StorageState,
		CreatedAt: r.CreatedAt, LastUsedAt: r.LastUsedAt, Evicted: r.Evicted,
	}
}
```
- [ ] **Step 4: 注入 + config**

在 `internal/config/config.go` 的 `BrowserConfig` 加：
```go
	SessionTTLSeconds  int `json:"session_ttl_seconds" yaml:"session_ttl_seconds"`   // 0 = 不回收
	ReapIntervalSeconds int `json:"reap_interval_seconds" yaml:"reap_interval_seconds"` // 0 = 默认 60s
```
（用 int 秒，避免引入 duration 解析；Runtime 侧转 `time.Duration`。）

在 `internal/cli/command.go` 构造 `browser.NewRuntime(...)` 处（Phase 1/2 已有 `sharedBrowser`），把 config 与 store 接上：
```go
	brt, err := browser.NewRuntime(browser.RuntimeConfig{
		Headless:     cfg.Browser.Headless,
		BinPath:      cfg.Browser.BinPath,
		SessionTTL:   time.Duration(cfg.Browser.SessionTTLSeconds) * time.Second,
		ReapInterval: time.Duration(cfg.Browser.ReapIntervalSeconds) * time.Second,
		Store:        newSQLiteBrowserStore(repo), // repo = 已打开的 *storage.SQLiteRepository
	})
```
（用调查步找到的实际 repo 变量名。若该处拿不到 repo，沿 `sharedBrowser` 构造链把 repo 传进来。`internal/app/app.go` 若也建 Runtime 且有 repo，同样注入；无 repo 则 `Store: nil`（纯内存，可接受）。）

- [ ] **Step 5: 跑 + 全量**

Run:
```
go test ./internal/cli/ -run TestSQLiteBrowserStoreBridge -v
go build ./... && go test ./... 2>&1 | tail -15
go test ./internal/runtime/ -run TestEveryProductionToolIsGateable
```
Expected: 适配器测试 PASS；全量绿（chromium-tag 跳过）；drift 绿。

- [ ] **Step 6: Commit**

```bash
git add internal/cli/browser_store.go internal/cli/browser_store_test.go internal/cli/command.go internal/config/config.go internal/app/app.go
git commit -m "feat(browser): wire SQLite-backed session persistence into runtime"
```

---

## 验证 Phase 3 DoD（对照 spec §8 Phase 3）

- [ ] storageState（cookies）落 SQLite `browser_sessions`（Task 1 往返 + Task 3 capture）
- [ ] 空闲会话按 TTL 回收：先落盘登录态、再释放 Context、置 nil（Task 4 `reapIdle` 选择 + `evictSession` 顺序）
- [ ] 回收后再访问自动重建 Context 并恢复登录态（Task 5 chromium：回收→重访→`sid=loginX` 保留）
- [ ] 进程重启后从盘加载会话（`NewRuntime` List+Adopt，Context=nil 懒重建）
- [ ] 字段级写穿：Touch 不清 storage_state（Task 1 `TestBrowserSessionTouchPreservesStorageState`）
- [ ] `Store=nil` 时纯内存，Phase 1/2 行为与全量测试不受影响（Task 2 nil-persist 测试 + 全量绿）
- [ ] drift 绿；`go test ./...` 全绿

---

## 已知近似与后续债务

| 近似 | 本 Phase 做法 | 后续 |
|---|---|---|
| storageState 内容 | 仅 cookies | localStorage/sessionStorage：per-origin CDP DOMStorage 快照 |
| 重建导航 | Open 重访时按传入 url（或持久 active_url）导航 | 多 tab：只恢复活跃 tab |
| 落盘时机 | 元数据 Create/Delete 写穿；storageState 仅回收/Close 抓 | 若需更强持久：动作后节流抓 cookies（避免每步 GetCookies 开销）|
| reaper | 单进程内存遍历 + 逐会话回收 | 服务模式多实例：加 DB 级租约/归属（Phase 13）|
| 重启 seq | Adopt 维护 seq 不撞号 | 若换 UUID 会话 id 则无需 |
| Touch 写穿接线 | 端口已备 `Touch`；动作路径调用点在 Task 5 Open/Read/Click/Type 里按需补（本 plan 端口+回收+重建为主，逐动作 Touch 可作收尾接入） | 若发现登录态刷新不及时，在成功动作后调用 `persist.Touch` |
```
