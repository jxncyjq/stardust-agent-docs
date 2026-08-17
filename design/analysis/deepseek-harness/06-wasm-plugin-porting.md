---
id: "analysis-dsh-wasm-porting-006"
title: "dsh 插件模型向 Go+WASM 的移植分析与选型"
aliases: ["dsh wasm porting", "Go WASM 插件选型", "legion wasm plugin"]
type: "analysis"
category: "design/analysis/deepseek-harness"
tags: ["deepseek-harness", "wasm", "wazero", "extism", "go-plugin", "legion", "plugin-architecture"]
version: "1.0.0"
created: "2026-08-15"
updated: "2026-08-15"
author: "jxncyjq"
status: "review"
parent: "analysis-dsh-index"
children: []
related_docs:
  - id: "analysis-dsh-plugin-system-003"
    relation: "extends"
    path: "./03-plugin-system.md"
  - id: "analysis-dsh-insights-005"
    relation: "extends"
    path: "./05-insights.md"
  - id: "analysis-dsh-capabilities-004"
    relation: "references"
    path: "./04-core-capabilities.md"
  - id: "analysis-dsh-index"
    relation: "related_to"
    path: "./index.md"
---

# dsh 插件模型向 Go+WASM 的移植分析与选型

<!-- @section: overview -->
## 概述

前提：Legion 计划以 **WASM** 实现插件系统（截至 2026-08-15，仓库内尚无 WASM 插件代码，属设计期）。本文回答两件事：

1. dsh（TypeScript / Cordis / 同进程）的插件设计，哪些能抄进 Go+WASM，哪些必须重做。
2. Go 侧 WASM 生态与其他插件机制的成熟度对比，以及 agent 场景该怎么选。

核心判断：**dsh 插件模型建立在「同进程 + 同语言 + 共享对象引用」之上**——`ctx.effect()` 返回闭包 disposer、waterfall 传 `next` 回调、Service 直接方法调用、scope key 按对象同一性比较。WASM 是跨边界、无共享内存、只能传序列化值。因此结论是：**抄它的契约纪律与失败语义，别抄它的闭包机制。**

调研数据来源与时效见文末「参考来源」，均为 2026 年公开资料。

<!-- @end-section -->

<!-- @section: portable -->
## 一、可直接移植（语义与宿主语言无关）

| dsh 机制 | WASM 下如何落 |
|---|---|
| **按 key 查服务 + `inject` 声明依赖** | guest manifest 声明 `requires: ["fs","shell"]`，host 按依赖拓扑激活。WASM 下比 dsh 更关键——guest 物理上 import 不到具体实现，只能拿 host function |
| **Capability Seam 三角色** | Service Definition = host function 组（ABI 契约）；Provider = host 原生模块**或** WASM 模块；Consumer = 调用方。三角色一起设计，不许只做一个 |
| **跨边界 id 必须 branded** | dsh 里是约定，WASM 里是物理必然。guest 只能持 `SessionId` / `ToolCallId` / `FileHandle` 这类不透明句柄，host 侧查表还原 |
| **工具契约四件套** | 强制 canonical output schema + 纯函数 `render(args,value)` + schema 白名单导出（`execute`/`timeout`/`presenter` 绝不进模型请求）+ 纯函数 `presentCall`/`presentResult`。**「纯函数」在 WASM 下是天然优势**：可确定性重放，实时流与日志回放共用一套渲染 |
| **模型可见 ⟺ 已落日志** | guest 无法私藏状态——它对模型说的每句话都必须经 host 落盘。WASM 隔离让这条从「纪律」变成「可强制」 |
| **无硬编码可调参数 + misconfig 响亮失败** | guest 配置是 schema 化字段，实例化前校验；缺依赖或字段非法直接拒绝加载 |
| **scope 两层扁平 + shadowing + restrict** | per-agent 工具集：scoped 注册**不向下继承给 subagent**；被 restrict 掉的工具**既不进 prompt 也拒绝执行**，与不存在不可区分 |
| **失败语义清单** | 审批缺席 fail-closed、能力不匹配响亮拒绝、锁最后释放留可检测孤儿锁、崩溃不截断长 turn、强制程度上报 full/partial |
| **生成物 + 新鲜度门禁** | ABI 契约、工具目录、能力图从源码生成，CI 校验。WASM 下更必要——**ABI 漂移是静默杀手** |

<!-- @end-section -->

<!-- @section: rework -->
## 二、必须改造（机制换掉，语义保留）

### 1. waterfall `next()` —— 最大改造点

dsh 的 around-middleware 靠传闭包，WASM 传不了函数。按拦截点分档：

| dsh 拦截点 | WASM 方案 |
|---|---|
| `tools/pre-execute`（可重排策略） | **可给 guest**。降级成 host 编排链：host 逐个调 `guest.pre_execute(exec) -> Decision{allow/deny/ask}`，短路 = 返回非 allow。放弃「包装参数再委派」，只保留「决策 + 可选改写」 |
| `guard()`（单调最终拒绝） | **可给 guest**。天然无 next，纯 `(exec) -> deny \| abstain` |
| `tools/execute`（around 包裹生命周期） | **不给 guest**。超时/重试/指标留 host 原生。硬要给就两段式 `begin(token)` / `end(token,result)`，但 signal 替换与词法闭包生命周期做不到，收益不抵复杂度 |
| `tools/post-execute`（结果改写） | **可给 guest**。`(exec,result) -> result'` |
| `tools/result`（只观察不可变结果） | **可给 guest**，最安全的挂点 |

原则：**guest 拿策略与变换，host 保留生命周期编排。**

### 2. `ctx.effect()` disposer → host 侧注册账本

guest 返回不了函数。改成：guest 每次注册（工具 / prompt 段 / 监听器）host 记一条 ledger 行，带 module instance id；模块卸载时 host 按 ledger 逆序回卷。**「注册即效果」语义保留，机制从闭包换成记账。** 这一步做对，热卸载与热重载自动成立——dsh 的 HMR 也就是靠这个。

### 3. scope key：对象同一性 → 显式 scope id

dsh 用「活体 agent 就是自己 scope 的 key」，按对象同一性比较。WASM 下改成 host 分配的 `ScopeId`，guest 注册时携带。

### 4. Typert → 从 Go 类型生成 ABI

dsh 从 TS `ts.Program` 生成 host/client 双向契约 + codec + 校验，并保留「源码模式弱回退（SRC）」。Legion 的对应物：从 Go 服务定义生成 ABI（protobuf 或 WIT）+ guest 绑定 + host dispatcher。**两档设计值得照抄**：开发期允许弱契约跑起来，正式构建必须严格生成。

### 5. spill / pruner 权重上升

dsh 里超大工具结果落盘返回定位符，目的是省 token。WASM 里**额外省跨边界拷贝**——大结果不进 guest 线性内存。优先级应比 dsh 更高。

<!-- @end-section -->

<!-- @section: wasm-only -->
## 三、WASM 独有、dsh 给不了答案的部分

| 问题 | dsh 现状 | Legion 需自研 |
|---|---|---|
| 资源计量 | 只有工具级协作式 timeout | **wazero 无 fuel 计量**，只能 context deadline；另需内存上限、实例池 |
| 能力授权 | `restrict()` + 工具 gateable 白名单（逻辑层） | **实例化时绑定 host function 白名单（物理层）**——比 dsh 强一档，应作为主机制 |
| 并发模型 | `isConcurrencySafe` opt-in + 共享内存 | 实例是否可重入、是否一调用一实例 |
| ABI 版本 | `SESSION_FORMAT_VERSION` 单调、无兼容承诺 | guest ABI 版本协商，比 dsh 更需要真兼容策略 |
| 边界校验 | 规则是「同进程类型化边界信任 TS，只在 wire/worker/进程边界校验」 | **每次 host↔guest 调用都是 wire 边界**，全部要校验 |

<!-- @end-section -->

<!-- @section: go-wasm-eco -->
## 四、Go 的 WASM 生态（2026 现状）

### 运行时层

| 运行时 | CGO | 性能（相对原生） | 成熟度 | 关键限制 |
|---|---|---|---|---|
| **wazero** | **无**（纯 Go） | 约 4.7x 慢，两年无明显变化 | Tetrate 全职团队维护，2026-05 仍在发版；Go 生态事实标准 | **仅 WASI p1，p2 未实现**；**无 fuel 计量**，超时靠 `context` + `WithCloseOnContextDone(true)` |
| **wasmtime-go** | 需要 | 约 2.4x 慢，逐年变快 | Bytecode Alliance，最规范 | CGO → 交叉编译、静态链接、构建矩阵复杂度全部上涨 |
| **wasmer-go** | 需要 | 2025 退步、2026 恢复；开 `wide_arithmetic` 为最快 | 次之 | CGO + 历史维护波动 |

**结论：Go 宿主选 wazero，除非性能是硬瓶颈。** 无 CGO 对 Legion 价值极高——要出 Windows/macOS/Linux 多平台且带 Wails GUI，引入 CGO 会显著恶化构建矩阵。

代价必须认清：**约 4.7x 原生开销 + 无 fuel 计量**。重计算插件会痛；防失控只能靠 context deadline，而它在「下一个操作边界」轮询，纯计算死循环必须开 `WithCloseOnContextDone` 才能及时中断。

### 插件框架层

| 方案 | 底座 | ABI 定义方式 | 语言支持 | 实战验证 |
|---|---|---|---|---|
| **Extism** | go-sdk 基于 wazero | 自有 ABI + manifest | 15+ host SDK / 7+ PDK（Go/Rust/JS/Python/C#…） | **Helm 4 选它做官方插件系统**，评价为「当今最成熟、支持最好的 Wasm 插件系统」 |
| **knqyf263/go-plugin** | wazero | **protobuf 定义接口 + 代码生成** | 仅 Go guest | Trivy 作者出品，schema 驱动、签名不易破 |
| **proxy-wasm-go-sdk** | 各家 proxy | proxy-wasm ABI | Go(TinyGo) | Higress / Envoy 系网关大规模生产 |
| 自研 ABI | wazero | 自定（JSON/protobuf over 线性内存） | 自定 | Traefik v3 WASM 插件走这条 |

Extism manifest 自带 `timeout_ms`、`memory.max_pages`、`allowed_hosts`、`allowed_paths`、`config`，外加 host functions——**能力授权与资源限制是声明式的**，正是「WASM 相对 dsh 的真优势」的现成实现。

### Guest 语言层

| 路径 | 状态 | 适用 |
|---|---|---|
| **标准 Go 1.24+ `GOOS=wasip1` + `//go:wasmexport` + `-buildmode=c-shared`** | 已 GA。reactor 模式：产物无 `_start`，只有 `_initialize` 与导出函数，可反复调用不重初始化 | **Legion 首选**：Go 栈零学习成本、标准工具链、完整 stdlib |
| **TinyGo `-target wasip2`** | 0.33+ 原生组件模型；`wit-bindgen-go` 生成 WIT 绑定 | 需要组件模型/WIT、需要小体积时 |
| Rust / JS / Python PDK | Extism 覆盖 | 让外部开发者写插件 |

> ⚠️ **标准 Go 编译器至今不支持 Component Model / WASI p2**，仅 TinyGo 支持；而 wazero 也尚未实现 p2。因此 **2026 年 Go+WASM 的可落地组合仍是 wasip1，不是组件模型**。要上 WIT 就得接受 TinyGo（stdlib 有缺）并换运行时。

<!-- @end-section -->

<!-- @section: go-plugin-mechanisms -->
## 五、Go 插件机制全景对比

| 机制 | 隔离 | 热插拔 | 崩溃影响宿主 | 跨语言 | 调用开销 | 分发 | 生产实证 |
|---|---|---|---|---|---|---|---|
| **编译期注册**（import + init） | 无 | 无 | 是 | 否 | 0 | 重编译 | 所有 Go 项目 |
| **stdlib `plugin`（.so）** | 无 | **不能卸载** | 是 | 否 | ~0 | 极差 | 几乎无人用 |
| **hashicorp/go-plugin**（子进程 + gRPC） | 进程级 | 有 | 否 | 是（任何 gRPC 语言） | **30–50µs/次** | 每平台一份二进制 | **Terraform / Vault / Nomad / Consul 多年** |
| **WASM（wazero + Extism / protobuf）** | **沙箱级 + 能力白名单** | 有 | 否 | 是 | 调用便宜、**拷贝贵** | **单文件跨平台** | Helm 4、Traefik v3、Higress |
| **嵌入 Go 解释器 yaegi** | 弱（同进程、可达宿主） | 有 | 是 | 否 | 慢 | 源码 | **Traefik 插件市场** |
| **嵌入脚本 Lua / Starlark / Risor / Expr** | 弱–中 | 有 | 一般否 | 否 | 中 | 源码 | 网关、规则引擎 |
| **MCP（进程外协议）** | 进程级 | 有 | 否 | 是 | stdio/网络往返 | 各自部署 | **agent 生态事实标准** |

三条硬事实：

1. **stdlib `plugin` 直接淘汰**：插件与宿主必须完全相同的 Go 版本、build tags、flags，否则运行时崩溃；**加载后不能卸载、不能替换**，热重载只能换文件名或重启。实践上要求「同一个人/同一套构建产出」——那不如编译进去。
2. **hashicorp go-plugin 的真成本不是 30–50µs，是分发**：插件是原生二进制，要出 N 个 OS×ARCH。面向终端用户装插件是灾难。
3. **yaegi 的隔离是假的**：解释器同进程运行、可访问宿主。Traefik 敢用是因为插件作者都写 Go 且经过市场审核；不可信第三方插件不能用它。

<!-- @end-section -->

<!-- @section: selection -->
## 六、agent 场景选型

agent 的「插件」实际是**四类不同东西**，塞进一个机制必然拧巴：

| 类别 | 例子 | 特征 | 选择 |
|---|---|---|---|
| **A 纯逻辑工具 / 策略** | 文本处理工具、pre-execute 权限策略、结果改写、prompt 段贡献、渲染投影 | 无 OS 需求、要沙箱、要热插拔、要开放第三方 | **WASM** ✅ |
| **B 重 OS 能力** | subprocess / PTY、浏览器接管、LSP 宿主、文件监听 | 需真实系统调用、长生命周期、大数据流 | **host 原生 Go**；必须解耦时才用 hashicorp go-plugin 子进程 |
| **C 生态互操作** | 用户自带第三方工具服务 | 已有标准、跨厂商 | **MCP** |
| **D 第一方核心** | agent loop、session log、工具注册表、审批 | 真相源，不可卸载 | **编译进去**，不做插件 |

### 推荐架构

```text
┌─ D 核心（编译期）: agent loop / session / 工具注册表 / 审批 / 沙箱执行器
├─ B 能力提供者（host 原生 Go）: fs / shell / subprocess / browser / lsp
│    ↑ 以 host function 形式向下暴露；能力白名单在实例化时绑定
├─ A 插件面（WASM，wazero）: 工具实现 + 策略钩子 + 渲染投影
└─ C 外部生态（MCP client）: 第三方 MCP server → 转成内部工具注册
```

### A 层选型：wazero + Extism

> **后续修正（2026-08-16）**：本节的 Extism 推荐在 Legion 落地设计中被推翻。核实后发现 **Extism Go PDK 实际要求 TinyGo**（官方文档：TinyGo 是编译 Extism Go 插件的最佳方式；<0.34 不支持 Reactor），而 TinyGo 的 `reflect` 限制与 protobuf 强类型契约相冲突。Legion 最终采用**自研精简 ABI on wazero + 标准 Go guest**，权衡表与决策见 [[design-legion-plugin-system-001]] §6.1。下文保留原始生态评估——Extism 仍是当前最成熟的通用 WASM 插件框架，只是不适配「标准 Go guest + protobuf 契约」这一组约束。

理由：

1. Helm 4 这种量级项目完成选型后选了它，省掉自研 ABI 的坑。
2. `timeout_ms` / `max_pages` / `allowed_hosts` / `allowed_paths` 是**声明式能力授权**，正好对上 Legion 既有的 `toolauth.gateable` 白名单思路，且是**物理强制**而非逻辑约定。
3. 多语言 PDK：未来让用户用 Python/JS 写工具插件时不必换底座。

若更看重**类型安全的接口演进**（Legion 是 Go 强类型栈），备选 **knqyf263/go-plugin 风格：protobuf 定义 ABI + 代码生成**——更接近 dsh 的 Typert 做法，签名漂移在编译期就炸。两者可结合：Extism 做载体，protobuf 做 payload schema。

**Guest 首选标准 Go 1.24 wasip1 reactor**（`//go:wasmexport` + `-buildmode=c-shared`）：团队零新语言成本，stdlib 完整。等 wazero 支持 WASI p2 再考虑组件模型迁移——**现在别赌 WIT**。

### 必须提前设计的三点

1. **超时与失控**：wazero 无 fuel。每次插件调用必须带 `context.WithTimeout`，运行时开 `WithCloseOnContextDone(true)`，否则纯计算死循环拖死宿主 goroutine。
2. **拷贝成本**：调用便宜、**跨边界数据拷贝贵**。大工具结果必须走「落盘 + 返回定位符」（dsh 的 spill seam），别让 100KB 的 grep 结果进插件线性内存。
3. **能力最小化**：host function 白名单在**实例化时**绑定，不同插件不同集合。这比 dsh 的 `restrict()` 强一档——dsh 是逻辑过滤，WASM 能做到物理不可达。

### 不建议

- stdlib `plugin`（不能卸载 + 版本耦合）
- yaegi 承载不可信插件（同进程无隔离）
- 一上来就上 Component Model / WIT（wazero 无 p2、标准 Go 无 CM、只剩 TinyGo 且 stdlib 有缺）
- 用 hashicorp go-plugin 做面向终端用户的插件分发（多平台二进制地狱）；它只适合**自有**重组件的解耦

<!-- @end-section -->

<!-- @section: conclusion -->
## 七、一句话结论

**wazero + Extism（或 protobuf ABI）跑纯逻辑插件，host 原生 Go 提供 OS 能力并以 host function 下发，MCP 接外部生态，核心编译进去。** dsh 的契约纪律（seam 三角色、注册即效果、模型可见即落日志、工具四件套、scope 两层扁平）继续作为语义规范；waterfall `next()`、对象同一性 scope key、同进程信任边界这三条必须重做。

<!-- @end-section -->

<!-- @section: references -->
## 参考来源

运行时与生态（2026 年公开资料）：

- [wasm-runtime-comparison (2026)](https://github.com/wasmruntime-io/wasm-runtime-comparison)
- [Performance of WebAssembly runtimes in 2026 — Frank Denis](https://00f.net/2026/06/23/webassembly-runtimes-2026/)
- [wazero community / maintenance](https://wazero.io/community/) · [wazero pkg.go.dev](https://pkg.go.dev/github.com/tetratelabs/wazero)
- [Wazero Hardening for Go Embedders](https://www.systemshardening.com/articles/wasm/wazero-hardening/)

插件框架：

- [H4HIP: Helm 4 Wasm plugin system](https://helm.sh/community/hips/hip-0026/)
- [Extism Go SDK](https://github.com/extism/go-sdk) · [Extism Manifest 文档](https://extism.org/docs/concepts/manifest/)
- [knqyf263/go-plugin README](https://github.com/knqyf263/go-plugin/blob/main/README.md?plain=1)
- [Traefik v3 WASM 插件深入](https://traefik.io/blog/traefik-3-deep-dive-into-wasm-support-with-coraza-waf-plugin)
- [Higress WASM Go 插件开发](https://higress.ai/en/docs/latest/user/wasm-go/)
- [API Gateway Plugin Architectures Compared (2026)](https://zuplo.com/learning-center/api-gateway-plugin-architectures-compared)

Guest 工具链：

- [Extensible Wasm Applications with Go (go.dev)](https://go.dev/blog/wasmexport) · [Go 1.24 Release Notes](https://tip.golang.org/doc/go1.24)
- [Compile Go to Wasm Components with TinyGo and WASI P2 — wasmCloud](https://wasmcloud.com/blog/compile-go-directly-to-webassembly-components-with-tinygo-and-wasi-p2/)

其他插件机制：

- [Go plugin package](https://pkg.go.dev/plugin) · [hashicorp/go-plugin](https://pkg.go.dev/github.com/hashicorp/go-plugin) · [go-plugin-benchmark](https://github.com/uberswe/go-plugin-benchmark)
- [MCP 2026 Roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/)

<!-- @end-section -->

## 相关文档

- [[analysis-dsh-plugin-system-003|dsh 插件系统]] — 被移植的原始设计
- [[analysis-dsh-insights-005|评估与借鉴]] — 可移植性三档评估
- [[analysis-dsh-capabilities-004|核心能力]] — spill / 工具管线等被引用的机制
- [[analysis-dsh-architecture-002|系统架构]]
- [[analysis-dsh-overview-001|项目总览]]
- [[analysis-dsh-index|DeepSeek Harness 分析索引]]
