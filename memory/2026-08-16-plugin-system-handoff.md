---
id: "reference-plugin-system-handoff-20260816-001"
title: "接续入口 — 插件系统 / prompt cache 未完事项（2026-08-16）"
aliases: ["插件系统接续", "plugin system handoff", "prompt cache 未完事项", "2026-08-16 handoff"]
type: "reference"
category: "memory"
tags: ["handoff", "worklog", "plugin-system", "wasm", "prompt-cache", "deepseek", "legion-agent"]
version: "3.1.0"
created: "2026-08-16"
updated: "2026-08-17"
author: "jxncyjq"
status: "archived"
parent: null
children: []
related_docs:
  - id: "reference-plugin-p1-handoff-20260817-001"
    relation: "superseded_by"
    path: "./2026-08-17-plugin-p1-handoff.md"
  - id: "design-legion-plugin-system-001"
    relation: "references"
    path: "../design/architecture/legion-plugin-system.md"
  - id: "bug-prompt-cache-backend-mismatch-001"
    relation: "references"
    path: "../agents/bug/2026-08-16-prompt-cache-backend-mismatch.md"
  - id: "analysis-dsh-wasm-porting-006"
    relation: "references"
    path: "../design/analysis/deepseek-harness/06-wasm-plugin-porting.md"
---

# 接续入口 — 插件系统 / prompt cache 未完事项

<!-- @section: overview -->
## 这份文档是什么

> 📦 **历史存档（2026-08-17 起）。** 其中 A1/A2/B/C 已全部完成，剩余事项见 [[reference-plugin-p1-handoff-20260817-001|2026-08-17 接续入口]]——**新会话请从那里开始**。本文保留完整的原始范围与完成情况，便于对照。

2026-08-16 一整轮工作的收尾。

那一轮做了三件事：分析 deepseek-harness（dsh）的插件架构 → 为 Legion 设计 WASM 插件系统 → 用 spike 验证关键假设，过程中意外发现 prompt cache 的后端错配问题并顺手修掉了其中一环。

**已交付**（都已落盘 / 已提 PR）：

| 产出 | 位置 | 状态 |
|---|---|---|
| dsh 架构分析 6 篇 + 索引 | `docs/design/analysis/deepseek-harness/` | 完成 |
| Legion 插件系统设计方案 | `docs/design/architecture/legion-plugin-system.md` | 完成（§6 已按实测更新） |
| prompt cache 后端错配取证 | `docs/agents/bug/2026-08-16-prompt-cache-backend-mismatch.md` | 完成，含真机实测 |
| P0 生命周期内核 TDD 计划 | `legionAgent/docs/superpowers/plans/2026-08-16-plugin-lifecycle-kernel.md` | 已执行完成 |
| 能力目录分区渲染 TDD 计划 | `legionAgent/docs/superpowers/plans/2026-08-16-capability-render-partition.md` | 已执行完成 |
| 分区渲染实现 | PR [#79](https://github.com/jxncyjq/stardust-agent-server/pull/79) | ✅ 已合并 |
| P0 生命周期内核 + 4 个既有缺陷修复 | PR [#80](https://github.com/jxncyjq/stardust-agent-server/pull/80) | **已提，待合并** |
| P0.5 能力面契约（§6.12） | `docs/design/architecture/legion-plugin-system.md` | ✅ 定稿，2/4 已落地 |
| Kimi 缓存实测 | bug 文档 §Kimi 实测 | ✅ 完成，两次独立复现 |

<!-- @end-section -->

<!-- @section: decisions -->
## 已定的技术决策（不要重新讨论）

| 决策 | 结论 | 依据 |
|---|---|---|
| 插件载体 | **WASM**（第三方开发 + 分发 + 热加载三个需求都成立，只有 WASM 同时满足单文件跨平台 + 真沙箱 + 能力最小化） | `06-wasm-plugin-porting.md` §六 |
| WASM 运行时 | **wazero**（纯 Go 无 CGO，保住多平台构建矩阵与 Wails 打包） | 实测：4.7x 原生开销可接受 |
| guest 语言 | **Rust（推荐）+ 标准 Go**。**TinyGo 已于 2026-08-17 排除**，不作选项也不再评估 | spike S1/S2 实测 |
| 契约格式 | **JSON + JSON Schema**（放弃 protobuf——跨语言场景下编译期类型安全拿不到，且与 Legion 既有 `tool.Descriptor.InputSchema` 同构） | 见设计方案 §6.2 |
| Component Model / WASI p2 | **移除，不是「以后再评估」** | 标准 Go 编译器不支持（`GOOS=wasip2` = golang/go#65333，Open + Backlog + 无 PR），wazero 也未实现 p2——双重阻塞 |
| Cordis 第三问（依赖收敛） | **要实现**，简化三态（Active / Suspended / Unloaded） | 早期曾判断「WASM 免疫所以不需要」，**这个判断是错的**：guest 不持有 host 引用只免除了「清理旧引用」，不免除「依赖不满足时不该运行」 |

<!-- @end-section -->

<!-- @section: verified -->
## 已用 spike 验证的事实（别再重测）

产物在会话 scratchpad 的 `spike/`（`abi/`、`size/`、`deps/`），可复跑。

| 项 | 结果 |
|---|---|
| 自研 ABI 跨语言 | Go + Rust guest 共用同一 host，host 约 **190 行** |
| 产物体积 | Rust **68KB** vs 标准 Go **3.26MB**（同功能：JSON + host function） |
| 调用开销 | Rust **9–11µs** vs Go **62µs**；编译 Rust **17ms** vs Go **651ms** |
| 最低内存 | Rust 4MiB 够用；Go **8MiB 起**（4MiB 下第 692 次调用 OOM） |
| CPU 失控 | wazero **无 fuel**，但 `WithCloseOnContextDone(true)` 实测能打断纯计算死循环（501ms）。**代价：模块被 close，实例报废不可复用** |
| 内存炸弹 | `WithMemoryLimitPages` 有效，1ms 内 trap |
| 依赖收敛 + 热加载 | 12 个用例 + 5 项变异验证全过（含三级级联挂起、热替换时在途调用、激活失败回滚） |
| wazero WASI p2 | **不支持**（v1.12.0 的 `imports/` 只有 `wasi_snapshot_preview1`） |

<!-- @end-section -->

<!-- @section: todo -->
## 未完事项

A1 / A2 / B / C 已完成（2026-08-17）。下表保留全部条目并标注结果，便于对照原始范围。

### A. 插件系统实施（主线）

| # | 事项 | 入口 | 说明 |
|---|---|---|---|
| ~~A1~~ | ~~P0 生命周期内核~~ | — | ✅ **已交付** PR #80。`lifecycle.Ledger`、可撤销注册、调用时解析的视图、owner 绑定，四个 task 全过含 `-race` |
| ~~A2~~ | ~~P0.5 能力面契约~~ | 设计方案 §6.12 | ✅ **四条契约定稿**：`call_tool` 双计数器、gateable 并集、审计归因、任务边界生效。其中后两条**已实现**（同 PR #80） |
| A3 | P1 WASM 宿主 | 设计方案 §6 | **未开始**。wazero 宿主 + ABI v1 + guest SDK + 实例池 + 分步激活回滚 + 在途收敛。前置（A1/A2）已就位 |
| A4 | P2 Loader + 依赖收敛 | 设计方案 §5.4 / §5.5 | **未开始**。三态收敛 + 热加载 + 任务边界生效。spike 已验证机制可行 |
| A5 | P3 分发面 | 设计方案 §9 | **未开始**。签名、OCI/HTTP 来源、`legion plugin` CLI、**GUI 授权同意流**（第三方场景的准入条件，最容易被忽略） |
| A6 | P4 作者体验 | — | **未开始**。多语言模板、dev 模式、示例、文档 |

**A1 修掉的三个现存缺陷**（与插件无关，当时就有）：

- `tool.Registry` 只有 `Register`，没有 `Unregister`、不返回 disposer → 现在返回幂等 revoke，重名 panic
- `Subset()` / `Without()` 是快照拷贝 → 现在是持父引用的视图，父级撤销即刻对所有派生生效
- `Registry` 无锁 → 现在有 `RWMutex`

### B. prompt cache 收尾

| # | 事项 | 入口 | 说明 |
|---|---|---|---|
| ~~B1~~ | ~~合并 PR #79~~ | — | ✅ 已合入 master（`1c9026d`） |
| ~~B2~~ | ~~落盘缓存命中 token~~ | — | ✅ **复核后发现早已接线**（当时判断有误）：`cachedTokens()` 同认扁平与 OpenAI 嵌套两种约定 → `resp.CachedTokens` → `st.cachedTokens` → `audit_events` / `conversation_turns` 的 `cached_tokens` 列。线上真实值查库即可 |
| ~~B3~~ | ~~`StablePrefixLen` / `cache_control` 去留~~ | — | ✅ **已定：保留**。复核后发现它本来就是 per-profile 显式 opt-in（`maas.profiles.*.prompt_cache`，默认 false），不是悬空中间态。实测结论已写进 `internal/config/config.go` 字段注释 |
| ~~B4~~ | ~~Kimi 缓存机制核实~~ | bug 文档 §Kimi 实测 | ✅ **已实测，两次独立复现逐 token 一致**。结论出乎意料——见下 |
| B5 | 插件变更只在任务边界生效 | 设计方案 §6.12 契约 4 | **未做**。契约已定稿，实现随 A4 |

**B4 的意外结论：Kimi 与 DeepSeek 的缓存语义不同。**

| 项 | DeepSeek | Kimi |
|---|---|---|
| usage 字段 | `prompt_cache_hit_tokens`（扁平） | `prompt_tokens_details.cached_tokens`（OpenAI 约定） |
| 尾部追加首发 | 2048 / 2093（98%） | 1792 / 2003（89%） |
| **中段插入首发** | **0**——部分前缀不计分 | **768**——**部分前缀计分** |
| `cache_control` | 接受并忽略 | 接受，但自动缓存本就全命中，无法证明其起作用 |

「两个后端语义相同」的推断只对了一半。**跨后端不能互推缓存语义，每个后端都得自己测。** 分区渲染在两者下都有收益，DeepSeek 是全有全无，Kimi 是部分损失。适配器已同时覆盖两种字段约定，无需改代码。

### C. 与既有机制接线

**全部处理完毕（PR #80）。** 取证过程中发现 C1 的真实形态比预想更严重——它今天就在生效，不必等插件。

| # | 事项 | 结果 |
|---|---|---|
| ~~C1~~ | loop guard 数错工具名 | ✅ **修复**。真实形态：`lazy_tools` 下每个工具都包在 `call_tool` 里，guard 记的是包装器名 → per-tool 熔断**退化成全局熔断**（任意 30 次调用即切断），且熔断事件与失败警告都在指责 `call_tool`。改为记录 `guardedToolName(call)`。插件接线时复用同一函数 |
| ~~C2~~ | 审计归因 | ✅ **实现**。`audit_events.origin` 列（`agent` / `delegate:depth-N` / `plugin:<name>`），走 context 传播，迁移回填既有行。变异验证通过 |
| ~~C3~~ | gateable 漏网 | ✅ **实现**。`toolauth.Contribute()` 动态登记，`IsGateable` 改为静态 ∪ 动态；重名或遮蔽内建 panic |
| ~~C4~~ | `toolLoopCap` 硬编码 | ✅ **复核后判定不改**。它的注释已写明是有意的 hard ceiling；而「批量工具被误切」的真实原因就是 C1，已随之消失 |
| ~~C5~~ | Windows 路径绕过 | ✅ **修复，且是实测出来的**。15 种拼写打到 guard 上：`..` 穿越 / 绝对外部 / 大小写 / 正斜杠 / 尾点尾空格**都已正确拒绝**（我原先怀疑的大小写与 8.3 短名被证伪）；真漏洞是 **`NUL` `CON` `COM1` `LPT1` `con.txt` 全部 `err=nil` 放行**（写到设备）与 **`notes.txt:hidden` ADS 放行**。修复走 GOOS 隔离 |

### D. spike 未覆盖

| 项 | 说明 |
|---|---|
| JS / Python guest | 未测 |
| 长时间 soak | 仅到 2000 次调用，不足以判断标准 Go guest 的 GC 碎片化 |
| 插件事件订阅开销 | 未测（高频事件对主链路的延迟影响） |

这几项都不阻塞 A3：Rust guest 已实测可用（68KB / 9µs / 4MiB），标准 Go guest 也实测可用（3.26MB / 62µs / 8MiB）。

> **TinyGo 已排除（2026-08-17 决定）**，从待办中移除：不再评估、不再补测，`.wasm` 体积成为瓶颈时也不回头找它。

<!-- @end-section -->

<!-- @section: lessons -->
## 五条教训（会影响下一轮的判断）

1. **先确认后端，再谈缓存语义。** 看到 `cache_control` 就假设后端是 Anthropic，导致连续两轮给出方向相反的结论。任何涉及缓存的判断，第一步查 `agent.json` 的实际 profile。

2. **「WASM 免疫依赖问题」是错的。** guest 拿不到 host 对象引用，只免除了「卸载后旧引用还在被调用」（幽灵），不免除「依赖不满足时插件仍在运行」（半残）。Cordis 的第三问要实现，只是可以简化成三态。

3. **只有首次发送能说明缓存问题。** 重复发送命中的是它自己刚写入的条目，与被测变量无关。第三轮探针把 repeat 算进对比，得出「两者相同」的错误结论；第四轮只测首发才拿到真数字（2048 vs 0）。同理，**变异验证是必须的**——本轮两次抓出「测试绿但其实没测到东西」。

4. **跨后端不能互推缓存语义。** 教训 1 之后仍犯了同类错误：确认了后端是 DeepSeek/Kimi，就假设两者语义相同。实测显示 Kimi 给部分前缀匹配计分，而这正是 DeepSeek 被实测否定的行为。每个后端都得自己测。

5. **安全边界要用探针实测，不要靠推理列风险。** C5 原本列的怀疑（大小写 / 8.3 短名 / UNC）**全部被证伪**——guard 早就正确处理了；真漏洞是没想到的两类（Windows 设备名、ADS）。推理出来的风险清单和实测出来的漏洞清单，交集可以是空的。

<!-- @end-section -->

<!-- @section: start -->
## 建议的起手式

**A1 / A2 / B / C 都已完成**，剩下的是 A3–A6 与 B5。

**如果只做一件事：合并 PR #80**，然后写 A3（P1 WASM 宿主）的实施计划。前置条件全部就位：生命周期内核可用，四条接线契约定稿，ABI 与依赖收敛都有 spike 实证。

**A3 的范围**：wazero 宿主 + ABI v1（§6.3）+ `pkg/legionplugin` guest SDK + 实例池 + 分步激活回滚 + 在途收敛。参考 spike 产物 `spike/abi/` 与 `spike/deps/`（host 约 190 行，12 测 5 变异）。

**A4 紧跟 A3**：Loader + 三态收敛 + 任务边界生效（B5 随之完成）。

**A5/A6 可以等**：分发签名与作者体验只在真有第三方插件时才有价值。

<!-- @end-section -->

## 相关文档

- [[design-legion-plugin-system-001|Legion 插件系统设计方案]] — 主设计文档，§6 是 WASM 插件机制
- [[bug-prompt-cache-backend-mismatch-001|BUG — prompt cache 断点机制与实际后端不匹配]] — 含真机实测数据
- [[analysis-dsh-wasm-porting-006|dsh 插件模型向 Go+WASM 的移植分析与选型]] — 生态调研与选型依据
- [[analysis-dsh-index|DeepSeek Harness 分析索引]] — dsh 架构分析 6 篇的入口
