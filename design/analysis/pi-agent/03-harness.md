---
id: "analysis-pi-harness-003"
title: "pi-agent Harness 分层与下一代 AgentHarness 规范"
aliases: ["pi harness", "AgentHarness", "pi lane operation"]
type: "design"
category: "design/analysis/pi-agent"
tags: ["pi-agent", "analysis", "harness", "durability", "state-machine", "recovery"]
version: "1.0.0"
created: "2026-08-15"
updated: "2026-08-15"
author: "jxncyjq"
status: "review"
parent: "analysis-pi-index"
children: []
related_docs:
  - id: "analysis-pi-architecture-002"
    relation: "extends"
    path: "./02-architecture.md"
  - id: "analysis-pi-insights-007"
    relation: "related_to"
    path: "./07-insights.md"
---

# pi-agent Harness 分层与下一代 AgentHarness 规范

<!-- @section: what-is-harness -->
## 「Harness」在 pi 里指什么

pi 的 system prompt 自称 *"operating inside pi, a coding agent harness"*。在这个仓库里，harness = **把模型、工具、会话存储、生命周期钩子缝合成一个可运行、可恢复的执行体**的那一层。

仓库里同时存在两代实现，读代码时必须分清：

| | 现役 harness | 下一代 `AgentHarness` |
|---|---|---|
| 位置 | `coding-agent/src/core/agent-session.ts` + `agent/src/agent.ts` + `agent-loop.ts` | `packages/agent/src/harness/**` |
| 规范 | 无独立规范，散在代码注释 | `packages/agent/docs/harness.md`（2941 行实现规范） |
| 状态 | 生产可用（v0.84.x） | **脚手架**：`agent-harness.ts` 绝大多数方法 `throw HarnessNotImplemented`，`create()` 遇到已有记录直接抛 |
| 已落地子模块 | — | `session/`（JSONL/memory/树视图）、`reducer.ts`、`tools/`、`skills.ts`、`compaction/`、`telemetry.ts`、`prompt-templates.ts` |
| 并发单位 | 单会话单流 | **lane**（车道），一个会话多条并行车道 |
| 持久化 | 会话 JSONL + 内存状态 | 三仓存储 + 操作状态机寄存器，崩溃可续 |

**结论先行**：想学「怎么把 Agent 跑起来」看现役；想学「怎么让 Agent 崩了能续」看规范。后者是 pi 目前最有参考价值的设计资产，也是 Legion 最缺的一块。
<!-- @end-section -->

<!-- @section: current-harness -->
## 现役 harness 的能力面

`AgentHarness`（规范版）的公开接口 `AgentLane` 已经写好，可以当作现役能力的清单来读：

```ts
interface AgentLane {
  prompt(text | message[]): Promise<RunResult>
  skill(name, additionalInstructions?): Promise<RunResult>       // 直接以技能启动一次运行
  promptFromTemplate(name, args?): Promise<RunResult>
  compact(options?): Promise<CompactionResult>
  navigateTree(targetId, options?): Promise<NavigationResult>    // 树导航 + 可选分支摘要
  resume(): Promise<ResumeResult>                                // 崩溃/挂起恢复
  abort(): Promise<AbortResult>                                  // 返回被排空的 steer/followUp
  steer / followUp / nextRun / cancelQueued                      // 三条队列
  recordUsage(usage, options?)                                   // 用量账本
  peekAction() / executeAction() / runToCompletion()             // 手动驱动模式
  getModel/setModel, getThinkingLevel/setThinkingLevel
  getActiveTools/setActiveTools
  watch(): WatchHandle<LaneSnapshot>
}
```

两个细节值得单独指出：

1. **`skill()` 和 `promptFromTemplate()` 是一等公民**——技能不是「模型自己想起来去 read 的文件」的唯一形态，也可以由宿主直接发起一次以技能内容为提示词的运行。
2. **`drive: "automatic" | "manual"`**：`peekAction()/executeAction()` 把循环拆成可单步执行的动作（`ActionInfo` 枚举了 `append_entry`/`stream_assistant`/`execute_tool`/`hook`/`sleep` 等）。**测试和确定性回放**因此变得可能——这是把 loop 写成状态机而不是 `while` 的直接红利。
<!-- @end-section -->

<!-- @section: three-stores -->
## 下一代设计：三仓存储

规范 §0.3 把所有持久化收敛为三种形态，这是整份文档的公理：

```
entries        会话树           write-once、append-only、不可变
registers      当前可变状态      命名空间化的类型化 cell，覆写或删除
usage ledger   成本历史         append-only 行
```

派生不变式（Part 9）里最锋利的几条：

- **「每份 payload 只存在于一个地方」**：entry、register、ledger 三选一，没有第四处能藏数据。
- **「配置与编排永不进入树」**：删掉所有 `op.*` 和 `pending.entry` 寄存器后，必须还剩一份完整合法的会话与账本。
- **「热路径不得靠折叠历史推断状态」**：没有历史可折叠，执行/恢复/分支必须走索引。
- **register 删除就是删除**，没有墓碑；JSON `null` 只在某命名空间类型允许时才是合法值。
- entry 的 parent 链**永不改变**，分支共享前缀，什么都不复制。

对比现役实现（JSONL 追加 + 内存状态 + 重启重放）：三仓模型把「可变状态」显式隔离到 registers，于是**崩溃恢复不需要重放整条历史**，只要读五个寄存器加有限 hydration（规范 R1 slice 的验收项之一就是「restore without history reads」）。
<!-- @end-section -->

<!-- @section: lanes -->
## Lane：会话内的并行车道

```
session = 条目树 (entry tree)
        + 事实 (facts，可变命名空间 KV：会话名、条目标签、应用自定义)
        + 车道 (lanes，指向树的命名游标)
        + 用量账本
```

一条 lane 拥有：自己的 leaf、模型配置、三条队列、**至多一个 operation**。每个会话必有 `main`。

动机写得很直白：Slack 线程、子 Agent、共享历史上的并行工作。这解决了现役实现「一个会话只能有一条活跃流」的限制——而这正是 Legion 多会话/多任务场景会撞上的同一堵墙。

`LaneBusy` 是一个显式错误类型（携带 `lane`/`operationId`/`operationKind`），说明**并发控制是接口级契约**，不是内部实现细节。
<!-- @end-section -->

<!-- @section: operation-fsm -->
## Operation：把 Agent 循环写成状态机

一次「被接受的车道工作」= 一个 operation，三种 intent：`run` / `compaction` / `navigation`。

- `op.meta/{id}` 写一次（不变元数据）
- `op.state/{id}` **每次转换整体覆写**（总状态，即「程序计数器」）
- 终结事务原子地删除两者并写 `lane.lastResult`

```ts
type RunPhase =
  | { kind: "checkpoint"; continuation; triggerEntryId; thresholdCheckedTriggerEntryId?; skipInboxOnce? }
  | { kind: "assistant"; generation }
  | { kind: "tools"; batch }
  | { kind: "compaction"; reason: "threshold" | "overflow"; structural; resumeAfter: CheckpointPhase }
  | { kind: "deferred"; deferred }
  | { kind: "failure_drain"; error; provenance }
```

设计要点：

- **没有 `finished` 成员**。操作结束就没有状态，结果落在 `lane.lastResult`。用类型系统消灭「已完成但状态还在」的中间态。
- `settings`（压缩配置、队列模式、工具执行模式）在**接受时原子快照**，后续 setter 只影响下一个 operation。运行中改配置不会撕裂当前运行。
- 队列项只存 entry id，payload 放 `pending.entry/{id}` 寄存器；`Control.cancel_requested` 里记 `drainedSteer/drainedFollowUp` 的 id，其 pending 寄存器**在排空后仍存活**，只由终结事务删除——这是「abort 后消息不丢」的机制。
- `overflowRecoveryUsed` 标志防止「上下文溢出 → 压缩 → 又溢出」的死循环；消费新用户输入时复位。
<!-- @end-section -->

<!-- @section: tools-fsm -->
## 工具批的持久化语义（§3.8）

每个工具调用在状态机里走 `planned → effect_pending → completed`，每一步都是一次事务：

| 转换 | 事务内容 |
|------|----------|
| planned → dispatch | 写 `op.tool_args/{opId}:{stepId}:{i}` = 生效后的参数，状态置 `effect_pending, replay` |
| effect_pending → completed | 插入结果 entry、更新 `lane.leaf`、插入工具用量行、状态置 `completed, terminate` |
| planned → completed（被拦/参数非法） | 插入**合成错误结果 entry**，不写 `tool_args`，不执行副作用 |
| 全批完成 | 折叠进最后一次结算，同时删除该批所有 `op.tool_args` 寄存器 |

配套的 `AgentTool` 新增字段：**`replay?: "never" | "safe"`**（默认 `never`）。恢复时，一个处于 `effect_pending` 的工具调用能否重放，取决于工具自己声明的幂等性。`before_tool` 在已 `effect_pending` 的调用上**不会重跑**，`after_tool` 只在 safe replay 上重跑。

这是整份规范里**最值得 Legion 直接抄的一条**：把「工具是否可重放」变成工具定义的一部分，而不是恢复逻辑里的 if-else。
<!-- @end-section -->

<!-- @section: hooks -->
## Hooks：11 个拦截点与聚合语义

规范 §5.6 定义的 hook 全集（harness 全局注册，payload 均带 `lane`）：

```
before_run · before_resume · before_run_end
transform_context · before_request · before_payload · after_response
before_tool · after_tool
before_compaction · before_navigation
```

统一语义（这一段是本规范信息密度最高的地方）：

- 处理器按**注册序**运行，各自看到前一个的输出；`messages` 追加，`systemPrompt` 替换。
- **抛异常 = 发 `handler_error`、跳过该处理器、其余继续；唯独 `before_tool` fail-closed，直接拦掉工具**。危险操作默认拒绝。
- **持久化的 hook 输出在继续执行前先提交**；「返回了」不等于「持久了」，提交前崩溃会重跑 hook。
- `before_tool` 的参数替换会**重新校验**；第一个 block 终结，后续处理器不再运行。
- `before_compaction/before_navigation` 在第一个 decline 或结果处停止；同时返回 decline 和结果算处理器错误，按抛异常处理。
- 事件（events）看到的是 **hook 之后的值**，被动监听者不能改。

另有一张「重放矩阵」表明确每个 hook 在 fresh / retry / resume 三种情形下是否重跑，并坦承 `before_run_end` 可能在同一边界重复触发——**「精确一次」是显式非目标**，需要幂等的处理器自带持久标记。这种对语义边界的诚实，比任何实现都值钱。
<!-- @end-section -->

<!-- @section: recovery -->
## 恢复模型

`SuspendedOperation` 描述一个可恢复的挂起：

```ts
{ lane, kind, id, startedAt,
  reason: "crash" | "deferred",
  prompt?, deferred?, aborting?,
  missing: { tools: string[]; models: string[] } }
```

`missing` 字段是点睛之笔：**恢复失败可能是因为「当时那个工具/模型现在不存在了」**（扩展被卸载、模型下架），于是有专门的 `MissingIdentities` 拒绝类型，而不是一个泛泛的 error。工具与模型的身份在**派发时解析**，不在恢复时假设。

`reason: "deferred"` 对应 provider 的异步/延迟响应（`DeferredHandle`），恢复时每次 `resume` 轮询一次，不做退避不做上限——因为轮询不产生新成本。
<!-- @end-section -->

<!-- @section: build-order -->
## 交付顺序（Part 8）说明了它的工程观

规范把实现切成 **Track S（存储/搜索/开发 TUI，可并行）** 与 **Track R（运行时，严格串行，全程只跑 Memory 后端）** 两条互不阻塞的轨道，共 4 + 12 个 slice。每个 slice 的验收标准写死：

> 实现其命名行为的端到端路径 + 针对「正常路径、引入的每个状态、拥有的每个崩溃边界、拥有的每个竞态的两种顺序」的聚焦测试，并通过 `npm run check`。
> **若实现暴露出设计矛盾、缺失转换或明显更简方案，停下来送审——不得在 slice 内部悄悄发明新的持久契约。**

另外两条罕见但正确的态度：

- 「`packages/agent/src/harness/**` 及其全部测试在 slice 1 可**直接删除**」——不为已有代码支付兼容成本。
- 「**已有测试是证据，不是权威**」——保留断言未变行为的，其余随被删代码一起删。
<!-- @end-section -->

## 相关文档

- [[analysis-pi-architecture-002|运行时架构与主循环]]
- [[analysis-pi-tools-004|工具系统与装载机制]]
- [[analysis-pi-insights-007|对 Legion 的启示]]
- [[analysis-pi-overview-001|项目架构总览]]
