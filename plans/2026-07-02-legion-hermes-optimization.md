# Legion 参考 Hermes 的功能与 Token 优化 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 参考 hermes-agent v0.17 的成熟机制，为 legionAgent 补齐 token 地基（准确计数、prompt 缓存、压缩调优）与四项能力（增强 delegation、MoA 多模型、session_search、Curator 技能生命周期）。

**Architecture:** legionAgent 是 Go/DDD 运行时，主循环 `internal/runtime/runtime.go::RunTask` 基于**单 prompt 字符串**驱动 `port.MaasInferenceClient.Generate`，工具循环把 `basePrompt`（稳定）+ `toolCtx`（易变）拼成一个 prompt 逐轮重发。本计划遵守既有 **Fail-Loud 铁律**（禁止 fallback/静默吞错，见 `legionAgent/CLAUDE.md`），改造尽量增量、接口先行、TDD。

**Tech Stack:** Go 1.x、SQLite（`internal/storage/sqlite.go`，需启用 FTS5）、`internal/port` 接口层、`internal/cognitive` 压缩、`internal/adapter` MaaS 适配、`internal/skill` 技能。

---

## 前置说明（务必先读）

1. **非 git 仓库**：当前 `F:\source\stardust\Legion` 不是 git 仓库。每个 Task 末尾的 `git commit` 步骤是逻辑检查点；执行前需先在 `legion/legionAgent` 或仓库根 `git init`，否则把 commit 步骤当作「运行完整验证门」的标记。
2. **验证门**（每个 Task 完成后必跑，见 `legionAgent/CLAUDE.md`）：
   ```bash
   cd legion/legionAgent && go build ./... && go vet ./... && go test ./... && gofmt -l .
   ```
   全绿且 `gofmt -l .` 为空方算完成。Windows 上注意 `.gitattributes` 已固定 `*.go` 为 LF。
3. **Fail-Loud**：所有新代码错误路径必须 `return fmt.Errorf("<动作> <标识>: %w", err)` 或领域错误，禁止返回零值假装正常、禁止 `_ = err`。错误路径必须有测试断言。
4. **实现前先读的文件**（对应 Task 会再指明具体行）：
   - `internal/runtime/runtime.go`（主循环、`generate`、`inferenceTools`）
   - `internal/runtime/lazytools.go`（meta-tool 协议范式）
   - `internal/cognitive/compressor.go`（`countTokens`、4 层压缩）
   - `internal/port/ports.go`（`InferenceRequest/Response`、`MaasInferenceClient`）
   - `internal/adapter/http_maas.go` + `maas.go`（provider 适配）
   - `internal/skill/lifecycle.go` + `registry_sync.go`（技能状态机）
   - `internal/storage/sqlite.go`（schema、无 FTS5）

---

## Task 1：CJK 感知 Token 计数器（A1，P0）

**背景**：`internal/cognitive/compressor.go` 的 `countTokens(text) = len(strings.Fields(text))` 按空格分词。中文无空格→整段≈1 token，压缩阈值 `before <= TokenLimit` 失效。

**Files:**
- Create: `legion/legionAgent/internal/port/token.go`（`TokenCounter` 接口）
- Create: `legion/legionAgent/internal/cognitive/tokencount.go`（CJK 感知实现）
- Create: `legion/legionAgent/internal/cognitive/tokencount_test.go`
- Modify: `legion/legionAgent/internal/cognitive/compressor.go`（`countTokens`/`countHistoryTokens` 改走可注入 counter）

**Step 1: 写失败测试** — `internal/cognitive/tokencount_test.go`

```go
package cognitive

import "testing"

func TestCJKAwareCounter(t *testing.T) {
	c := NewCJKTokenCounter()
	cases := []struct {
		name string
		in   string
		min  int // 下界断言，避免脆弱精确值
	}{
		{"ascii_words", "the quick brown fox", 3},
		{"chinese_no_space", "服务端是唯一真相源结算消耗判定", 8},   // 旧算法会得 1
		{"mixed", "调用 list_tools 获取工具 schema", 6},
		{"empty", "", 0},
	}
	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			got := c.Count(tc.in)
			if got < tc.min {
				t.Fatalf("Count(%q) = %d, want >= %d", tc.in, got, tc.min)
			}
		})
	}
}

func TestCJKCounterBeatsWhitespaceOnCJK(t *testing.T) {
	c := NewCJKTokenCounter()
	cjk := "这是一段没有空格的中文文本用于压缩阈值判断"
	if c.Count(cjk) < 10 {
		t.Fatalf("CJK text under-counted: got %d", c.Count(cjk))
	}
}
```

**Step 2: 跑测试确认失败**
Run: `cd legion/legionAgent && go test ./internal/cognitive/ -run TestCJK -v`
Expected: FAIL（`NewCJKTokenCounter` undefined）

**Step 3: 定义接口** — `internal/port/token.go`

```go
package port

// TokenCounter estimates the token length of a piece of text. Implementations
// are heuristic; they exist to drive compression-threshold decisions, not
// billing (billing uses InferenceResponse.PromptTokens from the backend).
type TokenCounter interface {
	Count(text string) int
}
```

**Step 4: 实现 CJK 感知 counter** — `internal/cognitive/tokencount.go`

```go
package cognitive

import "unicode"

// CJKTokenCounter is a heuristic TokenCounter that approximates BPE token counts
// far better than whitespace splitting on CJK and code. ASCII runs are counted
// at ~4 chars/token (BPE average); each CJK ideograph counts as one token (a
// safe upper-ish bound so compression triggers rather than silently under-counts).
type CJKTokenCounter struct{}

// NewCJKTokenCounter returns a ready-to-use CJK-aware counter.
func NewCJKTokenCounter() *CJKTokenCounter { return &CJKTokenCounter{} }

// Count returns the estimated token length of text.
func (c *CJKTokenCounter) Count(text string) int {
	tokens := 0
	asciiRun := 0
	flush := func() {
		if asciiRun > 0 {
			tokens += (asciiRun + 3) / 4 // ceil(asciiRun/4)
			asciiRun = 0
		}
	}
	for _, r := range text {
		switch {
		case isCJK(r):
			flush()
			tokens++
		case unicode.IsSpace(r):
			flush()
		default:
			asciiRun++
		}
	}
	flush()
	return tokens
}

// isCJK reports whether r is a CJK ideograph or common CJK symbol range that a
// BPE tokenizer typically splits per-character.
func isCJK(r rune) bool {
	return unicode.Is(unicode.Han, r) ||
		unicode.Is(unicode.Hiragana, r) ||
		unicode.Is(unicode.Katakana, r) ||
		unicode.Is(unicode.Hangul, r)
}
```

**Step 5: 跑测试确认通过**
Run: `cd legion/legionAgent && go test ./internal/cognitive/ -run TestCJK -v`
Expected: PASS

**Step 6: 把 compressor 接到可注入 counter** — `internal/cognitive/compressor.go`

- 在 `ContextCompressorConfig` 加字段 `Counter port.TokenCounter`。
- `NewContextCompressor`：`if cfg.Counter == nil { cfg.Counter = NewCJKTokenCounter() }`（这是**合法默认**，非兜底：契约允许不传 counter）。
- 把 `countHistoryTokens`/`countTokens` 的调用改为 `c.cfg.Counter.Count(...)`。保留旧 `countTokens` 自由函数供其它调用点，或一并迁移（`grep -rn countTokens internal/`）。

**Step 7: 跑全 cognitive 测试**
Run: `cd legion/legionAgent && go test ./internal/cognitive/ -v`
Expected: PASS（若旧 `compressor_test.go` 依赖旧计数值，按新 counter 更新期望值）

**Step 8: 验证门 + Commit**
```bash
cd legion/legionAgent && go build ./... && go vet ./... && go test ./... && gofmt -l .
git add internal/port/token.go internal/cognitive/tokencount.go internal/cognitive/tokencount_test.go internal/cognitive/compressor.go
git commit -m "feat(cognitive): CJK-aware token counter for compression thresholds"
```

---

## Task 2：Prompt 缓存 —— InferenceRequest 稳定前缀（A2，P0）

**背景**：`RunTask` 每轮重发 `basePrompt + toolCtx`，`basePrompt` 跨轮稳定。要复用 Anthropic/兼容 provider 的 prompt 缓存，需让 adapter 知道「哪段是稳定前缀」并注入 `cache_control` 断点。

**Files:**
- Modify: `legion/legionAgent/internal/port/ports.go`（`InferenceRequest` 加 `StablePrefixLen int`）
- Modify: `legion/legionAgent/internal/runtime/runtime.go`（`generate` 传稳定前缀长度）
- Modify: `legion/legionAgent/internal/adapter/http_maas.go`（Anthropic 路径注入 cache_control）
- Test: `legion/legionAgent/internal/adapter/http_maas_test.go`

**Step 1: 写失败测试**（adapter 层，断言 Anthropic 请求体在稳定前缀处带 cache_control）

先读 `internal/adapter/http_maas.go` 确认 Anthropic 请求构造函数名与请求体结构，再照它的既有测试风格（`http_maas_test.go`）写：

```go
func TestAnthropicRequestInjectsCacheControlAtStablePrefix(t *testing.T) {
	// 构造一个带 StablePrefixLen 的 InferenceRequest，
	// 断言序列化出的 Anthropic messages 中，稳定前缀对应的 content block
	// 带 "cache_control": {"type": "ephemeral"}。
	// 具体断言字段名依 http_maas.go 实际请求结构填写。
}
```

**Step 2: 跑测试确认失败**
Run: `cd legion/legionAgent && go test ./internal/adapter/ -run TestAnthropicRequestInjectsCacheControl -v`
Expected: FAIL

**Step 3: 扩展 InferenceRequest** — `internal/port/ports.go`

```go
type InferenceRequest struct {
	RequestID string
	Prompt    string
	Tools     []InferenceTool
	Images    []string
	// StablePrefixLen marks how many leading runes of Prompt are stable across
	// calls in the same task (system + task framing). Adapters that support
	// provider prompt caching (e.g. Anthropic cache_control) place a cache
	// breakpoint at this boundary. Zero means "no known stable prefix" and is
	// fully backward compatible — adapters treat the whole prompt as volatile.
	StablePrefixLen int
}
```

> 契约声明 `StablePrefixLen=0` 为合法「未知」态 —— 这是**显式可选**，非兜底。

**Step 4: runtime 传稳定前缀**

`RunTask` 中 `basePrompt` 已知。改 `generate` 签名带 `stablePrefixLen int`；首轮 `len([]rune(basePrompt))`，工具轮同样传 basePrompt 的 rune 长度（basePrompt 不变）。`boundPrompt` 可能裁剪易变尾部，但稳定前缀在头部、不受影响；若 `boundPrompt` 触发且改动了头部则传 0（fail-safe 的**合法**处理，因为前缀已不稳定）。

**Step 5: adapter 注入 cache_control**

`http_maas.go` 的 Anthropic 分支：把 `Prompt[:StablePrefixLen]` 与其余拆成两个 content block，前者带 `cache_control: {type: ephemeral}`。非 Anthropic/不支持缓存的 provider 忽略该字段（保持 prompt 原样拼接）。**不得**静默丢弃 —— 若 provider 不支持就是正常忽略（契约允许）。

**Step 6-8: 跑测试 / 验证门 / Commit**
```bash
cd legion/legionAgent && go build ./... && go vet ./... && go test ./... && gofmt -l .
git add internal/port/ports.go internal/runtime/runtime.go internal/adapter/http_maas.go internal/adapter/http_maas_test.go
git commit -m "feat(adapter): prompt cache breakpoint via InferenceRequest.StablePrefixLen"
```

---

## Task 3：压缩策略调优 + memory 批量预算（A3）

**Files:**
- Modify: `legion/legionAgent/internal/cognitive/compressor.go`（阈值/策略默认值随 Task 1 counter 校准）
- 读 `internal/adapter/memory.go`（现有 memory 写入路径）后决定 batch 接口位置
- Test: 对应 `_test.go`

**Steps（TDD 同前）：**
1. 写测试：给 compressor 一段超阈值的 CJK 历史，断言 `FourLayer` 策略被触发且 `TokensAfter < TokensBefore`。
2. 实现：确认 `Summarizer` 注入后 `CompressHistory` 走 summarize 分支；按新 counter 设定合理 `TokenLimit`/`ProtectedHead`/`ProtectedTail`/`ToolResultMaxChars` 默认。
3. memory 批量：读 `internal/adapter/memory.go`，加 `Apply(ops []MemoryOp)`（add/replace/remove）**原子**对齐最终字符预算；测试断言「单独 add 溢出、但 remove+add 组合成功」。错误路径 fail-loud。
4. 验证门 + commit：`git commit -m "feat(cognitive): tune compression + atomic memory budget ops"`

---

## Task 4：session_search + FTS5（B3，双赢：能力 + 降 token）

**背景**：`internal/storage/sqlite.go` 无 FTS5；会话历史目前靠堆叠进 prompt。加检索工具后可「按需检索替代长历史」。

**Files:**
- Modify: `legion/legionAgent/internal/storage/sqlite.go`（建 FTS5 虚拟表 + 迁移 + 写入同步 + 查询方法）
- Create: `legion/legionAgent/internal/tool/session_search.go`（`session_search` 工具）
- Test: `internal/storage/sqlite_test.go`、`internal/tool/session_search_test.go`
- 读：`internal/tool/registry.go`、`internal/tool/builtin.go`（工具注册范式）、`internal/tool/web.go`（一个完整工具样例）

**Step 1: 先确认 FTS5 可用**
Run: `cd legion/legionAgent && go test ./internal/storage/ -run TestFTS5Available -v`（先写一个建 `CREATE VIRTUAL TABLE ... USING fts5(...)` 的测试）
Expected: FAIL/或报 "no such module: fts5" → 需确认 sqlite 驱动启用 FTS5（`modernc.org/sqlite` 默认含；`mattn/go-sqlite3` 需 build tag `sqlite_fts5`）。**先解决驱动 FTS5 支持**再继续。

**Step 2-3: FTS5 表 + 查询方法（storage）**
- 迁移建 `messages_fts` 虚拟表（`content`, `session_id`, `message_id` 等），随消息写入同步 upsert。
- 加方法：`SearchMessages(ctx, query, limit)`（discovery）、`ScrollMessages(ctx, sessionID, aroundID, window)`（scroll）、`BrowseSessions(ctx, limit)`（browse）。全部 fail-loud。
- 测试：插入若干消息，`SearchMessages("token 压缩")` 命中正确行；scroll/browse 各一个测试。

**Step 4-5: session_search 工具**
- 照 `internal/tool/web.go` 结构写 `session_search` 工具：三模式按参数推断（有 `query`→discovery；有 `session_id`+`around_message_id`→scroll；无参→browse）。
- 注册进 registry + builtin 工具集（读 `builtin.go` 看 `_HERMES_CORE_TOOLS` 等价物）。
- 工具测试：三模式各断言返回 JSON 结构正确、错误参数 fail-loud。

**Step 6: 验证门 + Commit**
```bash
git commit -m "feat(storage,tool): FTS5-backed session_search (discovery/scroll/browse)"
```

---

## Task 5：增强 Delegation —— delegate_task 工具 + 子 Runtime（B1）

**背景**：`agent_resolver.go` 能解析子 agent 配置，`tool/agent_message.go` 有 agent 通信，但无让模型主动派生子任务的工具。子 agent 独立上下文 → 只回传摘要 → 省父 token。

**Files:**
- Create: `legion/legionAgent/internal/runtime/delegation.go`（`RunSubTask`、并发/后台调度、深度/角色限制）
- Create: `legion/legionAgent/internal/tool/delegate_task.go`（`delegate_task` 工具，桥接到 runtime）
- Modify: `internal/runtime/runtime.go`（暴露子 Runtime 构造 / 复用 `NewRuntime`）
- 读：`internal/runtime/agent_resolver.go`、`internal/agentregistry/*`、`internal/tool/agent_message.go`
- Test: `internal/runtime/delegation_test.go`、`internal/tool/delegate_task_test.go`

**Step 1: 写失败测试（单个委派）**
```go
func TestRunSubTaskReturnsSummaryOnly(t *testing.T) {
	// 用 fake MaasInferenceClient：父 Runtime 委派一个子任务，
	// 断言：(a) 子任务用独立 prompt 上下文（不含父的 toolCtx）；
	//       (b) 返回给父的是子任务最终 Result 文本（摘要），非全过程。
}
```

**Step 2: 跑失败** → **Step 3: 实现 `RunSubTask`**
- 子 agent：`agent_resolver` 解析角色配置 → `NewRuntime(subCfg)` 独立实例 → `RunTask` → 取 `TaskRun.Result`。
- `leaf`（默认，禁止再委派：子 Runtime 不注册 `delegate_task` 工具）vs `orchestrator`（可再委派，`depth < max_spawn_depth`）。深度超限 fail-loud。

**Step 4: batch 并行**
- `RunSubTasks(ctx, tasks)`：`golang.org/x/sync/errgroup` + 信号量限 `max_concurrent`（默认 3）。任一子任务错误按契约决定「整体失败」还是「逐条回报」——推荐逐条回报结果+错误（供模型看到），但**调度层错误**（超并发/派生失败）fail-loud。
- 测试：3 个子任务并行、结果顺序稳定、一个失败不静默吞。

**Step 5: background**
- `delegate_task(background=true)`：立即返回句柄；子结果完成经 `EventBus.Publish` 回灌成新回合事件。注意进程内、父退出即失（文档标注，非 durable）。
- 测试：后台句柄返回 + 完成事件发布。

**Step 6: delegate_task 工具 + 注册**
- 工具参数：`goal`、`context`、`toolsets`、`role`、`background`、`tasks[]`（batch）。桥接到 runtime。
- 注意 lazy 协议下它是被 `call_tool` 调用的真实工具。

**Step 7: 验证门 + Commit**
```bash
git commit -m "feat(runtime,tool): delegate_task with batch/background/leaf-orchestrator"
```

---

## Task 6：MoA 多模型协作（B2）

**背景**：`MaasInferenceClient` 单模型。MoA = N 个 reference model 并行生成 + 聚合器综合。one-shot，避免每轮 token 爆炸。

**Files:**
- Create: `legion/legionAgent/internal/runtime/moa.go`（`MoACoordinator`）
- Create: `internal/runtime/moa_test.go`
- 读：`internal/adapter/maas.go`、`maas_profile.go`（多 model/profile 路由），`internal/port/ports.go`

**Steps（TDD）：**
1. 写测试：`MoACoordinator` 用 fake MaaS，3 个 reference model 并行 `Generate`，聚合器收到「带标签的各参考输出」并产出最终答复；断言并行发生（用可计数的 fake）+ 聚合 prompt 含 3 段标签块。
2. 实现：`Aggregate(ctx, task, refModels []ModelRef, aggregator ModelRef)`；`errgroup` 并行；聚合 prompt 拼装参考块。结合 MaaS 质量感知路由（按任务类型选档，读 `maas_profile.go`）。
3. 触发点：one-shot（工具/斜杠或配置开关），**非** `RunTask` 每轮默认开。
4. 错误路径：某 reference model 失败——按契约要么剔除该路（记 Warn 日志）要么整体失败；结算/聚合环节不得用空结果冒充。测试覆盖。
5. 验证门 + commit：`git commit -m "feat(runtime): Mixture-of-Agents multi-model coordinator"`

---

## Task 7：Curator 技能生命周期（B4）

**背景**：`internal/skill/lifecycle.go` 有 Enable/Disable/status/risk，gene 有 success_rate/count，但无使用统计 + stale/archive 自动化。

**Files:**
- Create: `legion/legionAgent/internal/skill/curator.go`（usage sidecar + 确定性扫描）
- Create: `internal/skill/curator_test.go`
- Modify: `internal/skill/lifecycle.go`（加 `Archive` 状态转换，若无）
- 读：`internal/skill/registry_sync.go`、`internal/skill/manager.go`、`internal/storage/sqlite.go`（skill/gene 持久化）

**Steps（TDD）：**
1. 写测试：一个 `created_by=agent` 且 `last_activity_at` 超 `stale_after` 的技能，跑 `Curator.Sweep()` 后状态变 `stale`；再超 `archive_after` → `archived`；**pin 的技能不变**；bundled/hub 技能不受影响；**从不删除**。
2. 实现 usage sidecar：`use_count`/`view_count`/`last_activity_at`（复用 gene 的 success_count 或新表）。`Sweep` 是**确定性、零 token** 的纯 Go 逻辑。
3. 可选 LLM 合并 pass：`consolidate` 默认 `false`（契约显式可选）；开启才调辅助模型。
4. 边界：`Sweep` 幂等、并发安全（若后台跑）。错误 fail-loud。
5. 验证门 + commit：`git commit -m "feat(skill): Curator deterministic lifecycle sweep (zero-token)"`

---

## 收尾 Task 8：全量验证 + 文档回链

1. 全量门：`cd legion/legionAgent && go build ./... && go vet ./... && go test ./... && gofmt -l .` 全绿。
2. 在 `docs/design/analysis/hermes/08-hermes-v017-updates.md` 的 `@section: legion-token` 表格「Legion 现状」列，把已落地项从「缺失」更新为「已实现（见本计划 Task N）」。
3. 若新增系统，按 `legionAgent/CLAUDE.md` 要求在 `docs/architecture/` 加对应 ADR（delegation / MoA / session_search / curator 各一），提交引用本计划。

---

## 依赖与顺序

```
Task1(token计数) ─┬─> Task3(压缩调优, 依赖 counter)
                  └─> (所有压缩相关判断更准)
Task2(prompt缓存) ── 独立, 可与 Task1 并行
Task4(session_search) ── 独立(需先解决 FTS5 驱动)
Task5(delegation) ── 独立
Task6(MoA) ── 依赖 Task2 完成更省 token, 但可独立开发
Task7(curator) ── 独立
```

建议执行顺序（按 ROI + 依赖）：**Task1 → Task2 → Task4 → Task3 → Task5 → Task6 → Task7 → Task8**。
