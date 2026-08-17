---
id: "reference-plugin-p1-handoff-20260817-001"
title: "接续入口 — 插件系统 P1 及剩余事项（2026-08-17）"
aliases: ["插件系统剩余", "P1 WASM 宿主接续", "plugin p1 handoff", "2026-08-17 handoff"]
type: "reference"
category: "memory"
tags: ["handoff", "worklog", "plugin-system", "wasm", "wazero", "abi", "legion-agent"]
version: "1.0.0"
created: "2026-08-17"
updated: "2026-08-17"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "reference-plugin-system-handoff-20260816-001"
    relation: "supersedes"
    path: "./2026-08-16-plugin-system-handoff.md"
  - id: "design-legion-plugin-system-001"
    relation: "references"
    path: "../design/architecture/legion-plugin-system.md"
  - id: "bug-prompt-cache-backend-mismatch-001"
    relation: "references"
    path: "../agents/bug/2026-08-16-prompt-cache-backend-mismatch.md"
---

# 接续入口 — 插件系统 P1 及剩余事项

<!-- @section: overview -->
## 这份文档是什么

**只写还没做的。** 已完成部分见 [[reference-plugin-system-handoff-20260816-001]]（历史存档）。

新会话从这里开始，不需要任何前序对话上下文。

### 当前状态

| 项 | 状态 |
|---|---|
| PR [#79](https://github.com/jxncyjq/stardust-agent-server/pull/79) 能力目录分区渲染 | ✅ 已合入 master（`1c9026d`） |
| PR [#80](https://github.com/jxncyjq/stardust-agent-server/pull/80) P0 生命周期内核 + 4 个缺陷修复 | **待合并**（9 commit，全绿） |
| P0.5 能力面契约 | ✅ 定稿于设计方案 §6.12，4 条中 2 条已落地 |

**第一步：合并 PR #80。** 下面所有工作都建立在它之上。

<!-- @end-section -->

<!-- @section: remaining -->
## 剩余事项

| # | 事项 | 规模 | 依赖 |
|---|---|---|---|
| **A3** | **P1 WASM 插件宿主** | 大（本文档主体） | PR #80 合并 |
| A4 | P2 Loader + 三态依赖收敛 + 任务边界生效（含 B5） | 中 | A3 |
| A5 | P3 分发面：签名、来源、CLI、GUI 授权同意流 | 中 | A4 |
| A6 | P4 作者体验：多语言模板、dev 模式、示例 | 小 | A3 |
| D | spike 补测：TinyGo / JS / Python guest / 长 soak / 事件开销 | 小 | 无，随时可做 |

D 组不阻塞任何事：Rust guest（68KB / 9µs / 4MiB）与标准 Go guest（3.26MB / 62µs / 8MiB）都已实测可用。TinyGo 只影响「Go 插件作者的产物体积」一条。**2026-08-17 明确决定跳过 TinyGo 实测**，不下载工具链。

<!-- @end-section -->

<!-- @section: a3 -->
## A3：P1 WASM 插件宿主（主线）

### 范围

| 组件 | 落点 | 说明 |
|---|---|---|
| wazero 宿主 | `internal/plugin/host` | 运行时、编译缓存、实例池 |
| ABI v1 | `internal/plugin/abi` | 4 个 guest 导出 + 6 个 host function |
| guest SDK | `pkg/legionplugin` | Go 插件作者用；Rust SDK 单独仓或 A6 |
| 能力白名单 | `internal/plugin/capability` | 未授权的能力**不注册进模块**（链接期缺失，非运行时拒绝） |
| 分步激活 + 回滚 | 复用 `lifecycle.Ledger` | 已就位，见下 |
| 在途调用收敛 | 实例池 `inflight sync.WaitGroup` | Draining → 在途归零 → Unloaded |

**不含**：`plugin.Loader` 与 `plugins.yaml`（A4）、签名与分发（A5）。P1 的验收是「手工指定一个 `.wasm`，能挂载、能被模型调用、能干净卸载」。

### 已就位的前置（PR #80 交付，直接用）

```go
// 所有权账本：逆序撤销、失败不互阻、幂等句柄
lifecycle.NewLedger() *Ledger
(*Ledger).Add(owner Owner, label string, dispose func() error) func() error
(*Ledger).DisposeOwner(owner Owner) error
(*Ledger).Snapshot() map[Owner][]string

// 可撤销注册（重名 panic；有意覆盖走 Replace）
(*tool.Registry).RegisterDescriptor(Descriptor, Handler) func()
(*tool.Registry).Replace(Descriptor, Handler) func()

// 一步到位：注册 + 记账
tool.RegisterOwned(ledger, owner, registry, descriptor, handler) func() error

// 动态 gateable 登记（重名或遮蔽内建 panic）
toolauth.Contribute(GateableTool) func()

// 审计归因（走 context 传播）
tool.WithCallOrigin(ctx, "plugin:<name>") context.Context

// loop guard 的有效工具名（插件侧复用同一函数）
runtime.guardedToolName(call domain.ToolCall) string
```

**插件注册一个工具的完整动作**是三件事挂在同一个 `lifecycle.Owner` 上：

1. `tool.RegisterOwned(ledger, owner, reg, desc, handler)`
2. `ledger.Add(owner, "gateable:"+name, func() error { toolauth.Contribute(...)(); return nil })`
3. host function 入口 `ctx = tool.WithCallOrigin(ctx, "plugin:"+name)`

漏掉第 2 步 = per-agent `disabled_tools` 够不到这个工具 = **授权绕过**。

### 必须遵守的契约（设计方案 §6.12，已定稿）

| # | 契约 | 状态 |
|---|---|---|
| 1 | `call_tool` 双计数器：per-plugin 递归深度 3 + **与模型共用** `toolNameGuard` 的 per-task 总预算 | **A3 要实现**。分开计数 = 给插件开绕过通道 |
| 2 | gateable = 静态内建 ∪ 动态贡献 | ✅ 已实现，A3 只需调用 |
| 3 | 审计必须归因到发起者 | ✅ 已实现，A3 只需标注 |
| 4 | 插件变更只在任务边界生效 | A4 实现 |

### ABI v1 形状（spike 已验证，Go + Rust guest 共用同一 host）

**guest 导出**：

```
plugin_alloc(size i32) -> i32       // host 回写数据前必须调它
plugin_free(ptr i32, size i32)
plugin_manifest() -> i64            // 自描述：name/version/provides，激活后交叉校验
plugin_call(op i32, ptr i32, len i32) -> i64
```

**返回值打包**：`i64` 高 32 位 = ptr，低 32 位 = len。

**host function**（模块名 `legion`）：`log` / `config_get` / `kv_get` / `kv_put` / `http_request` / `read_file` / `call_tool`。目录与授权见设计方案 §6.4。

**三条实现约束**（踩过的坑，别重蹈）：

1. **能力检查在 host 侧再做一次**。「没授权就不注册函数」不够——授权了 `http` 但目标域名不在 `allowed_hosts`，必须在 `http_request` 内部拒绝返回 `DENIED`。
2. **host function 之间不得互相调用**。wazero 已知问题：一个 host function 调另一个会丢失 memory 访问。公共逻辑抽成普通 Go 函数。
3. **host 回写数据给 guest 必须走 guest 的 `plugin_alloc`**，不能直接写 guest 线性内存。

### 关键 wazero 配置（实测确定）

```go
wazero.NewRuntimeConfig().
    WithCloseOnContextDone(true).      // 唯一能打断纯计算死循环的手段（wazero 无 fuel）
    WithMemoryLimitPages(pages)        // Rust guest 64 页(4MiB)够；标准 Go guest 需 128 页(8MiB)起
```

⚠️ **`WithCloseOnContextDone` 的代价**：超时会 close 整个模块，**实例报废不可复用**。实例池必须把它标记为 dead 并新建，不能放回池子。

### spike 产物（会话临时目录，随时可能被清理）

```
C:\Users\ADMINI~1\AppData\Local\Temp\claude\F--source-stardust-Legion\
  6a4c287a-a170-41e1-89a8-ac561b2cadd1\scratchpad\spike\
    abi/host/main.go          host 约 190 行，跑通 Go + Rust 两种 guest
    abi/guest-go/main.go
    abi/guest-rust/
    size/                     四个标准 Go wasip1 产物（体积对比）
    deps/loader.go            依赖收敛 + 热加载原型
    deps/loader_test.go       12 用例 + 5 变异，全过
    deps/guests/{prov,cons}
```

**A3 开工第一件事：如果这些文件还在，先拷进 `legionAgent/internal/plugin/` 作为起点。** 如果已被清理，本节的 ABI 形状 + 配置足以重建；`deps/` 的三态收敛逻辑要重写，但 §5.5 的状态表已足够。

### A3 的 TDD 验收（建议）

- 一个 `.wasm` 能挂载、注册工具、被 `Registry.Execute` 调到、卸载后 `ErrToolNotFound`
- 卸载后 `ledger.Snapshot()` 为空、`toolauth.IsGateable()` 转 false
- 激活中途失败 → 已登记项逆序回滚（**这个测试极易假绿**：见 §教训）
- 纯计算死循环被 deadline 打断，实例标记 dead 不回池
- 内存炸弹被 `WithMemoryLimitPages` trap
- 插件发起的 `call_tool` 撞递归深度上限，且计入 `toolNameGuard`
- 未授权能力的 host function 在模块里根本不存在（链接期）

<!-- @end-section -->

<!-- @section: invariants -->
## 不可违反的约束

| # | 约束 | 违反的后果 |
|---|---|---|
| I1 | `Registry` 的任何派生视图不得拷贝 handler，只能持父引用 | 幽灵 handler 回归（PR #80 刚消灭） |
| I2 | host 侧组件不得缓存插件返回的对象/闭包，只能缓存 id | 同上，且卸载后无法收敛 |
| I3 | 新增任何工具必须登记 `toolauth` gateable | per-agent 授权绕过 |
| I4 | 加固/鉴权改动必审**所有** in-process 消费者 | 踩过：4B 只测后端漏了 GUI，合 master 即坏 403 |
| I5 | 写穿必须字段级、先落盘后释放锁 | 踩过：全行 UPSERT 覆盖并发写 |
| I6 | 插件发起的工具调用照常经 `tool.Registry.Execute` | 绕过权限、审计、超时、人工审批门 |
| I7 | chromium 相关代码走 build tag 隔离，PAL 之外零 `GOOS` 判断 | drift-guard 强制 |

<!-- @end-section -->

<!-- @section: decisions -->
## 已定的决策（别重新讨论）

| 决策 | 结论 |
|---|---|
| 插件载体 | **WASM**。第三方开发 + 分发 + 热加载三个需求同时成立，只有它满足单文件跨平台 + 真沙箱 + 能力最小化 |
| 运行时 | **wazero**。纯 Go 无 CGO，保住多平台构建矩阵与 Wails 打包。4.7x 原生开销可接受 |
| guest 语言 | 不限。第三方推荐 Rust；Go 用 TinyGo（未实测） |
| 契约格式 | **JSON + JSON Schema**。放弃 protobuf——跨语言拿不到编译期类型安全，且与既有 `tool.Descriptor.InputSchema` 同构 |
| Component Model / WASI p2 | **移除，不是「以后再评估」**。标准 Go 不支持（golang/go#65333 Open + Backlog + 无 PR），wazero 也未实现——双重阻塞 |
| 依赖收敛 | **要做**，简化三态（Active / Suspended / Unloaded）。早期「WASM 免疫所以不需要」的判断已被推翻 |
| `toolLoopCap = 30` | **保持硬编码**。注释已写明是有意的 hard ceiling |
| `StablePrefixLen` / `cache_control` | **保留**。per-profile 显式 opt-in，默认 false，为 Anthropic 兼容后端预留 |

<!-- @end-section -->

<!-- @section: measured -->
## 已实测的事实（别重测）

### WASM 侧

| 项 | 结果 |
|---|---|
| 自研 ABI 跨语言 | Go + Rust guest 共用同一 host，host 约 190 行 |
| 产物体积 | Rust **68KB** vs 标准 Go **3.26MB**（同功能） |
| 调用开销 | Rust **9–11µs** vs Go **62µs**；编译 Rust 17ms vs Go 651ms |
| 最低内存 | Rust 4MiB 够；Go **8MiB 起**（4MiB 下第 692 次调用 OOM） |
| CPU 失控 | wazero 无 fuel，`WithCloseOnContextDone(true)` 能打断（501ms），代价是模块被 close |
| 内存炸弹 | `WithMemoryLimitPages` 有效，1ms 内 trap |
| 依赖收敛 + 热加载 | 12 用例 + 5 变异全过（三级级联挂起、热替换在途调用、激活失败回滚） |
| wazero WASI p2 | **不支持**（v1.12.0 只有 `wasi_snapshot_preview1`） |

### prompt cache 侧（两个后端语义**不同**）

| 项 | DeepSeek | Kimi |
|---|---|---|
| usage 字段 | `prompt_cache_hit_tokens`（扁平） | `prompt_tokens_details.cached_tokens`（OpenAI 约定） |
| 尾部追加首发 | 2048 / 2093（98%） | 1792 / 2003（89%） |
| **中段插入首发** | **0**——部分前缀不计分 | **768**——部分前缀计分 |
| `cache_control` | 接受并忽略 | 接受，但自动缓存本就全命中，无法证明其起作用 |

适配器 `cachedTokens()` 已同时覆盖两种字段约定，无需改代码。

<!-- @end-section -->

<!-- @section: lessons -->
## 教训（会影响下一轮判断）

1. **先确认后端，再谈缓存语义。** 看到 `cache_control` 就假设是 Anthropic 后端，导致连续两轮方向相反的结论。

2. **跨后端不能互推缓存语义。** 确认了后端是 DeepSeek/Kimi 之后，仍假设两者相同——实测显示 Kimi 给部分前缀计分，正是 DeepSeek 被否定的行为。

3. **只有首次发送能测缓存放置。** 重复发送命中的是它自己刚写入的条目。第三轮探针把 repeat 算进对比，得出「两者相同」的错误结论。

4. **安全边界要用探针实测，不要靠推理列风险。** Windows 路径绕过原本怀疑的三项（大小写 / 8.3 短名 / UNC）**全部被证伪**；真漏洞是没想到的两类（设备名 `NUL`/`CON`、ADS `file:stream`）。推理清单与实测清单的交集可以是空的。

5. **变异验证是必须的。** 本轮两次抓出「测试绿但根本没测到东西」：
   - `TestActivationFailureRollsBack` 假绿——`activate()` 的所有失败点都在第一次 `ledger.Add` 之前，回滚路径是死代码。加了清单交叉校验（在实例入册**之后**）才让回滚变活
   - `auditOrigin` 默认归一被去掉后往返测试立刻 FAIL——证明测试真咬住了

6. **「WASM 免疫依赖问题」是错的。** guest 拿不到 host 引用只免除「幽灵」（卸载后旧引用还在被调用），不免除「半残」（依赖不满足时插件仍在运行、仍被模型看见、每次都失败）。

<!-- @end-section -->

<!-- @section: env -->
## 环境备注

- `-race` 并行跑多包时 storage 会偶发 CGO sqlite 崩溃（`unexpected return pc`），**master 上同样出现**，与代码无关。逐包串行跑即可
- Go 编译器本身偶发 `cmd/internal/obj.pctofileline` nil deref，`go clean -cache` 后重跑
- docs 仓改动一直**未提交**（含更早遗留的未跟踪文件）。要提交需明确指示

<!-- @end-section -->

## 相关文档

- [[reference-plugin-system-handoff-20260816-001|接续入口 — 插件系统 / prompt cache 未完事项（2026-08-16）]] — 历史存档，含已完成部分的细节
- [[design-legion-plugin-system-001|Legion 插件系统设计方案]] — §5.5 三态收敛、§6 WASM 机制、§6.12 接线契约、§9 分期路线
- [[bug-prompt-cache-backend-mismatch-001|BUG — prompt cache 断点机制与实际后端不匹配]] — 两个后端的完整实测数据
- [[analysis-dsh-wasm-porting-006|dsh 插件模型向 Go+WASM 的移植分析与选型]] — 生态调研与选型依据
