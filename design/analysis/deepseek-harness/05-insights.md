---
id: "analysis-dsh-insights-005"
title: "DeepSeek Harness 评估与对 Legion 的借鉴"
aliases: ["dsh insights", "DeepSeek Harness 借鉴", "dsh vs legion"]
type: "analysis"
category: "design/analysis/deepseek-harness"
tags: ["deepseek-harness", "insights", "legion", "comparison", "architecture-decision"]
version: "1.0.0"
created: "2026-08-15"
updated: "2026-08-15"
author: "jxncyjq"
status: "review"
parent: "analysis-dsh-index"
children: []
related_docs:
  - id: "analysis-dsh-capabilities-004"
    relation: "depends_on"
    path: "./04-core-capabilities.md"
  - id: "analysis-dsh-architecture-002"
    relation: "depends_on"
    path: "./02-architecture.md"
  - id: "analysis-dsh-index"
    relation: "related_to"
    path: "./index.md"
---

# DeepSeek Harness 评估与对 Legion 的借鉴

<!-- @section: overview -->
## 概述

本篇是主观评估：dsh 哪些设计是真的强、哪些是它自己的包袱、以及哪些能落到 Legion（Go 技术栈的 agent server + GUI）上。

**免责**：Legion 侧的现状描述基于本仓已有文档与既往工作记录，落地前需按当时代码复核。

<!-- @end-section -->

<!-- @section: strengths -->
## 强在哪

### 1. 微内核不是口号，是可检查约束

大多数项目宣称插件化，最后都留一个「特权 core」。dsh 把它做成了门禁：

- 每个产品特性都必须映射到一条「监听器挂在已文档化扩展点上」的记录（cookbook 里那张表），**没有一行修改主循环**。
- 改主循环 ⇒ 必须同时更新 `docs/architecture.md`（AGENTS.md 硬约束）。
- 服务图、模块图、工具目录、配置目录、事件目录全部生成 + CI 新鲜度门禁，架构文档不会腐化成谎言。

### 2. 「模型可见 ⟺ 已落日志」这条不变式

一条规则解决了一大类问题：replay、fork、恢复、遥测、UI 回放、审计、压缩全部退化成同一条事件流的投影。代价是纪律——新增模型可见输入必须先新增 session 事件——但换来的是**任何一次模型请求都能从日志 + 代码重建**。

对比：多数 harness 里「prompt 里到底塞了什么」是靠日志字符串猜的。

### 3. Capability Seam 的三角色定义足够严格

关键在于它把「什么不是 seam」也定义清楚了：
- Service Definition 必须是 Cordis `Service`（抽象类或具体注册表），**不能是 TS `interface`**——因为 seam 要有运行时身份。
- seam 是完整能力，不是单个角色；只有当角色独立演化时才分包。
- 容器 / microVM / 远程执行是**兄弟 seam 实现**，不是 `ctx.sandbox` 的 provider。

这三条挡住了「接口越加越宽，最后每个 provider 都要实现自己不需要的方法」的经典退化。

### 4. Scope 的「两层、扁平」决策

per-agent 注册**不向下继承给 subagent**，子树行为一律用 lineage 数据表达。这是个反直觉但正确的选择：继承式作用域会让「这个 agent 到底能看到什么工具」变成不可推理的问题。配套的 `restrict()` 语义也很干净——被过滤掉的工具**既不出现在 prompt 也拒绝执行**，与不存在不可区分，杜绝了「模型看得见但调不动」的困惑。

### 5. 失败语义写得非常细

随手可举的例子：

- 审批缺席 ⇒ fail-closed 为 `unavailable`，不是放行。
- 沙箱强制程度是**被上报的事实**（`full` / `partial`），要求绝对边界的调用方必须自己处理差异，而不是假装 partial 等于 full。
- subagent 能力不匹配 ⇒ 响亮拒绝，绝不「接受后静默忽略」。
- compaction 锁最后释放 ⇒ 崩溃留下可检测的孤儿锁，而不是谎称完成的 end。
- 崩溃恢复**不截断**长 turn，补合成 `turn/end{interrupted}`，且 `interrupted` 是主循环永远不会发出的原因值——于是「这是恢复补的」可判定。

这些都是踩过坑才写得出来的条款。

### 6. 工具契约的完整度

强制 canonical output schema + 纯渲染投影 + 显式白名单的 schema 导出 + 纯函数 UI presenter + 显式并发 opt-in + 协作式超时。特别是「UI 渲染意图是工具设计的一部分，在设计期决定」这条，把「工具做完了再想怎么显示」的常见返工消灭在源头。

<!-- @end-section -->

<!-- @section: costs -->
## 代价与风险

| 项 | 说明 |
|---|---|
| **认知门槛陡** | 219 个包 + 一套自研框架词汇（seam / scope / carrier / shadowing / waterfall / effect）。新人不读完 primer + architecture + glossary 基本改不动东西 |
| **绑定 vendored Cordis** | 框架是 vendor 进来的固定源码副本（带 upstream SHA manifest 与同步流程）。好处是可控，坏处是升级要重新施加或退役本地修改并重跑全量测试 |
| **生成物依赖构建顺序** | Typert 必须先跑 Host 阶段生成契约，Client 才能编译；干净工作树无法跳过 Host 阶段。这是真实的开发摩擦 |
| **无兼容承诺** | 明确处于「pre-release，foundation over blast radius」阶段：后端拒绝旧的落盘格式，`SESSION_FORMAT_VERSION` 停在 `0`。作为参考架构可以抄，作为依赖要慎重 |
| **100% 逐文件覆盖率门禁** | 质量上是好事，但对贡献速度是显著税负 |
| **能力面广但深度参差** | E2B 组自己标注为 POC；部分 seam（如 `ctx.lsp` 只有四个规范化操作、无协议逃生舱）是刻意收窄的 |

<!-- @end-section -->

<!-- @section: legion -->
## 对 Legion 的对照与可移植点

Legion 与 dsh 的定位有重合（都是 agent harness + 自带 GUI），但技术栈与阶段不同：Legion 是 Go 服务 + Wails GUI，工具授权、会话落盘、浏览器接管、截断治理等已各自成型。以下按「是否值得抄」分档。

### 高价值、可直接借鉴

| dsh 机制 | 对 Legion 的意义 |
|---|---|
| **模型可见 ⟺ 已落日志 + 运行时不变式** | Legion 已有 `conversation_turns` / `audit_events`，但两者定位不同（一个对账、一个取证）。引入「凡进入模型请求的内容必须可从落盘记录重建」这条硬约束，并写一个启动期/回归期的断言，能一次性消灭「prompt 里到底塞了什么」类排查 |
| **`tool/call` 在执行前落盘** | 把「模型请求过什么」与「执行结果如何」拆成两条独立事实。Legion 排查工具循环事故时曾靠时间窗反推，前置落盘可以让这类取证从推理变成查询 |
| **拦截点选型规则（pre / guard / around / post / result）** | Legion 的 per-agent 工具授权、熔断/loop-cap、截断治理目前分散在不同层。用「可重排策略 = pre-execute；单调不变式 = guard；生命周期包裹 = around；结果改写 = post；只观察 = result」这套分类重排一遍，能解决「加新工具忘记登记 gateable」这类反复出现的漏网 |
| **审批 fail-closed + 沙箱强制程度上报** | Legion 的权限/沙箱路径值得补上「缺席即拒绝」与「partial 强制必须上抛」两条语义 |
| **溢出/剪枝分层（spill-policy 在 post-execute，pruner 在压缩之前）** | Legion 的截断治理三阶段已解决「谁截断、截断多少」，但缺一个「超大结果落盘 + 返回定位符 + 取回提示」的正式 seam。dsh 的 `ctx.spillStore` 是可直接照搬的形状 |
| **投影缓存的冷读阶梯** | `sessionProjectionCache` 的「缓存行 + 持久化尾部重放」让会话列表不必加载完整日志。Legion 的会话列表/标题/统计如果还在扫全量记录，这是成本极低的优化 |
| **subagent 能力协商** | start-time 能力静态广告 + 不匹配即响亮拒绝，比「尽力而为」健壮得多；Legion 若要接多种委派后端（in-process / 外部 CLI / 远程）应先定这套协商 |

### 中等价值、需权衡

| dsh 机制 | 权衡 |
|---|---|
| **Profile / Bundle 补丁层** | 概念优雅，但依赖 Cordis Loader 的 include/patch 能力。Legion 的 `agent.json` + per-agent 配置已能覆盖多数场景；只有在需要「第三方分发一整套能力组合」时才值得投入 |
| **Scope + shadowing** | Legion 已有 per-agent 工具授权（`disabled_tools`）。若要做 per-agent 人格/工具变体，可借鉴 shadowing 的「最具体者胜」与 restrict 的「过滤掉 = 既不可见也不可执行」语义，但不必引入完整 scope carrier 机制 |
| **Code Mode（`run_code`）** | 能显著压缩多工具调用轮数（Legion 的多轮 token 放大问题正源于此）。但它要求一个安全的代码运行时与「子调用同样穿过完整管线」的实现，工程量不小 |
| **Typert 式类型图生成 RPC** | Legion 的 Wails 绑定已提供类型化通道，收益不如 TS 单语言栈明显 |

### 低优先级 / 不建议照搬

- **219 包的粒度**：Go 生态的包边界成本与 TS 不同，强行拆细只会增加摩擦。
- **逐文件 100% 覆盖率门禁**：对当前阶段的 Legion 是过度约束。
- **vendored 框架**：Legion 没有等价的插件框架依赖，不必制造一个。

<!-- @end-section -->

<!-- @section: takeaways -->
## 三条最值得记的结论

1. **架构文档能不腐化，靠的是生成 + 门禁，不是自觉。** dsh 把服务图、模块图、工具目录、事件目录全部生成并在 CI 校验新鲜度——这是任何规模的项目都能抄的低成本做法。
2. **「不是什么」和「是什么」同样重要。** seam 的定义、scope 的两层扁平、sandbox 只管文件效果、Remote 只处理一元调用——每条都明确划掉了一片诱人的滑坡。
3. **失败语义要在设计期定死。** fail-closed、响亮拒绝、可检测孤儿锁、不截断长 turn、强制程度上报——这些条款的存在与否，决定了系统在真机上是「可排查」还是「玄学」。

<!-- @end-section -->

## 相关文档

- [[analysis-dsh-capabilities-004|核心能力]] — 本篇结论的证据来源
- [[analysis-dsh-architecture-002|系统架构]]
- [[analysis-dsh-plugin-system-003|插件系统]]
- [[analysis-dsh-overview-001|项目总览]]
- [[analysis-dsh-index|DeepSeek Harness 分析索引]]
