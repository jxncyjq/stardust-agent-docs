---
id: "analysis-hermes-updates-008"
title: "Hermes v0.13→v0.17 新增功能分析"
aliases: ["hermes updates", "hermes v0.17", "Hermes新增功能"]
type: "analysis"
category: "design/analysis/hermes"
tags: ["hermes-agent", "updates", "moa", "kanban", "curator", "delegation", "analysis"]
version: "1.0.0"
created: "2026-07-02"
updated: "2026-07-02"
author: "jxncyjq"
status: "review"
parent: "analysis-hermes-overview-001"
related_docs:
  - id: "analysis-hermes-index"
    relation: "parent"
    path: "./index.md"
  - id: "analysis-hermes-runtime-002"
    relation: "related_to"
    path: "./02-agent-runtime.md"
  - id: "analysis-hermes-tools-003"
    relation: "related_to"
    path: "./03-tools-skills-plugins.md"
  - id: "analysis-hermes-insights-006"
    relation: "related_to"
    path: "./06-hermes-insights.md"
---

<!-- @section: overview -->
# Hermes v0.13 → v0.17 新增功能分析

## 文档目的

`01`–`07` 分析文档基于 **2026-05-04** 的 Hermes Agent 快照撰写。此后 hermes-agent 演进到 **v0.17.0**（`pyproject.toml`），主线累计数千次提交，新增了一批**系统级**能力。本文档记录这些**增量**功能，保留 `01`–`07` 原快照的完整性，只在此集中补充。

- 分析对象版本：hermes-agent **v0.17.0**（对比基线：2026-05-04 快照）
- 素材来源：`skills/autonomous-ai-agents/hermes-agent/SKILL.md`（官方 curated v0.13–v0.17 摘要，权威）、`git log` 特性提交、`AGENTS.md`
- 覆盖范围：Agent 运行时（MoA、增强委派）、多 Agent 协作（Kanban）、技能生命周期（Curator）、会话控制（Goal、中途注入、检查点）、durable 系统（增强 Cron）、检索（session_search）、记忆（可插拔后端 + 批量操作）、隔离（Profiles）、网关（Relay/drain）、新增交互面（proxy/ACP/desktop）、安全（smart 审批/PII 脱敏）

## 新增功能总览

| # | 功能 | 分类 | 对 Legion 参考价值 | 关联旧文档 |
|---|------|------|------------------|-----------|
| 1 | **MoA（Mixture of Agents）** | Agent 运行时 | ★★★★★ 多模型协作/投票 | [[02-agent-runtime\|02]] |
| 2 | **增强 Delegation** | 多 Agent | ★★★★★ batch/background/orchestrator | [[06-hermes-insights\|06]] |
| 3 | **Kanban 工作队列** | 多 Agent | ★★★★☆ durable 多 worker 协作 | — |
| 4 | **Curator 技能生命周期** | 技能系统 | ★★★★☆ 零成本维护 | [[03-tools-skills-plugins\|03]] |
| 5 | **Goal 系统 + 中途注入** | 会话控制 | ★★★★☆ 跨回合目标/steer | [[02-agent-runtime\|02]] |
| 6 | **Checkpoints / rollback** | 会话控制 | ★★★☆☆ 文件系统快照 | — |
| 7 | **增强 Cron** | durable 系统 | ★★★☆☆ script/context_from/workdir | [[04-gateway-cli-deployment\|04]] |
| 8 | **session_search（FTS5）** | 检索 | ★★★★★ 零成本历史检索 | [[05-data-models\|05]] |
| 9 | **记忆增强** | 记忆系统 | ★★★★☆ 批量操作/字符预算 | [[02-agent-runtime\|02]] |
| 10 | **Profiles 隔离实例** | 运维 | ★★★☆☆ 多实例隔离 | — |
| 11 | **Automation Blueprints** | 运维 | ★★★☆☆ 无 cron 语法自动化 | — |
| 12 | **Relay / Gateway drain** | 网关 | ★★★☆☆ 多平台/优雅停机 | [[04-gateway-cli-deployment\|04]] |
| 13 | **proxy / ACP / desktop** | 交互面 | ★★★☆☆ 新增 surface | [[04-gateway-cli-deployment\|04]] |
| 14 | **smart 审批 / PII 脱敏** | 安全 | ★★★★☆ aux-LLM 风险分级 | [[05-data-models\|05]] |

<!-- @end-section -->

<!-- @section: moa -->
## 1. MoA — Mixture of Agents（多模型协作）

`02` 描述的主循环是**单模型调用**。v0.17 引入 MoA：多个**参考模型（reference models）**各自对同一输入产生输出，再由一个**聚合器（aggregator）**综合成最终答复。

- **触发**：`/moa` 斜杠命令，one-shot 模式；预设切换走 model picker。
- **可见性**：参考模型输出以带标签的分块渲染在最终聚合结果之前（CLI / TUI / desktop 均支持）。
- **状态**：参考模型能看到完整工具状态；在每次 user / tool 响应时触发。

**对 Legion 的意义**：`06` 已提出「Agent 循环支持多模型协作 + 投票」作为改进方向。Legion **已实现**（本计划 Task 6 + D4：`runtime.MoACoordinator` 参考模型并行 + 聚合器综合、全失败 fail-loud，经 `moa_consult` 工具触发，profile 名解析构建 ModelRef），可做**质量感知的多模型投票**（研究用一档、代码用另一档，聚合器裁决）。

<!-- @end-section -->

<!-- @section: delegation -->
## 2. 增强 Delegation（子代理委派）

`06` 中 `delegate_task` 还是「简单子 Agent 调用」。v0.17 显著增强：

- **单个 / 批量**：`delegate_task(tasks=[...])` 并行跑多个子任务，受 `delegation.max_concurrent_children`（默认 3）限制。
- **后台**：`delegate_task(background=true)` 立即返回句柄，父循环继续；子结果完成后作为新回合重新进入对话。
- **角色**：`leaf`（默认，不能再委派）vs `orchestrator`（可再生 worker，受 `delegation.max_spawn_depth` 限制）。
- **隔离上下文**：子 Agent 有独立 context + terminal session，独立 `IterationBudget`（`delegation.max_iterations` 默认 50）。
- **非 durable**：后台子进程仍是进程内的，父进程退出即丢失；要 outlive 用 cron 或 `terminal(background=True)`。

**对 Legion 的意义**：这是 Legion 「多 Agent 协作」（`06` §6.2）最直接的参考。Legion **已实现**（本计划 Task 5：`delegate_task` batch/background + leaf/orchestrator 深度限制）。子代理**独立上下文**同时是 token 优化手段——子任务的中间过程不污染父上下文，只回传摘要（见 [[08-hermes-v017-updates#token\|§token]]）。

<!-- @end-section -->

<!-- @section: kanban -->
## 3. Kanban — 多 Agent 工作队列

全新的 durable SQLite 看板，用于多 profile / 多 worker 协作。

- **驱动**：用户用 `hermes kanban <verb>`（init/create/list/show/assign/link/comment/complete/block/unblock/archive/tail…）。
- **worker 工具集**：dispatcher 派生的 worker 看到聚焦的 `kanban_*` 工具集（`kanban_show`/`complete`/`block`/`heartbeat`/`comment`/`create`/`link`），由环境变量 `HERMES_KANBAN_TASK` 门控。普通会话零 `kanban_*` schema 占用。
- **Dispatcher**：默认跑在 gateway 内（`kanban.dispatch_in_gateway`）——回收失效 claim、提升 ready 任务、原子 claim、派生指定 profile。连续 spawn 失败达 `failure_limit`（默认 2）自动 block。
- **隔离**：board 是硬边界（worker 环境固定 `HERMES_KANBAN_BOARD`）；tenant 是软命名空间。

**对 Legion 的意义**：对应 `06` §6.2「Agent 团队 + 工作流引擎 + 人机协作」。**worker 专属工具集门控**是 token 优化范式：不同角色只加载自身需要的工具 schema。

<!-- @end-section -->

<!-- @section: curator -->
## 4. Curator — 技能生命周期管理

`03` 描述了技能安装 / 渐进式加载，但没有生命周期治理。v0.17 补上 Curator：后台维护 **agent 自建技能**。

- **作用域**：只碰 `created_by: "agent"` 的技能；捆绑 + hub 安装的技能免疫。**从不删除**，最狠只归档。pin 的技能豁免一切自动流转。
- **机制**：追踪使用量（`~/.hermes/skills/.usage.json`：`use_count`/`view_count`/`patch_count`/`last_activity_at`/`state`/`pinned`），标记闲置为 stale，归档 stale，跑前留 tar.gz 备份。
- **成本**：确定性的闲置 / prune 扫描**零 token**；aux-model「合并重叠技能」pass **默认关闭**（`curator.consolidate: true` 显式开启）。

**对 Legion 的意义**：Legion **已实现**（本计划 Task 7 + D1：`skill.Curator.Sweep` 确定性零 token 扫描，workspace 技能 stale→archived、pinned/registry 免疫、从不删除；`Consolidate` LLM 治理为可选开关，默认关；SQLite `ListSkills` + 后台 `skill-curator-sweep` job 已接线，`System.WithUsage` 在技能选中时 `UsageStore.Touch` 记录活跃、与 Curator 共享同一实例，数据链路闭合）。此分层设计同样适用于 agent 自建技能 / Wiki 知识条目，避免知识膨胀。

<!-- @end-section -->

<!-- @section: session-control -->
## 5. Goal 系统 + 中途注入 + 检查点

一组围绕**长时会话可控性**的新能力：

### Goal 系统
`/goal [text]` 设定一个**跨回合的常驻目标**，Hermes 持续朝它推进直到达成（子命令 status/pause/resume/clear）。

### 中途注入（不破坏 prompt 缓存）
- `/steer <prompt>`：在下次工具调用后注入一条消息，**不中断**当前执行。
- `/queue <prompt>`：排队到下一回合。
- `/background <prompt>`：后台跑。
- `/busy [queue|steer|interrupt]`：控制「工作中按 Enter」的行为。

关键约束：**消息角色严格交替**、**对话中期绝不改系统提示/工具集**——这是保 Anthropic prompt 缓存的铁律（`AGENTS.md`「Never break prompt caching」）。

### Checkpoints / rollback / snapshot
- `--checkpoints` 开启文件系统检查点；`/rollback [N]` 恢复；`/snapshot` 存/恢复 Hermes 配置/状态快照。config：`checkpoints.max_snapshots`（默认 50）。

**对 Legion 的意义**：中途注入 + 角色交替 + 缓存不变式，是 prompt caching 的协议约束。Legion 已补 prompt 缓存断点（本计划 Task 2：`InferenceRequest.StablePrefixLen` + adapter `cache_control`，`boundPrompt` 裁头即传 0 保前缀不变式，见 §token）；中途注入 / 角色交替若后续引入须一并遵守这些约束。

<!-- @end-section -->

<!-- @section: cron -->
## 6. 增强 Cron

`04` 覆盖了基础定时。v0.17 增强 per-job 能力：

- **schedule 形态**：duration（`30m`/`2h`）、"every" 短语、5 字段 cron、ISO 时间戳。
- **per-job 旋钮**：`skills`、`model`/`provider` 覆盖、`script`（跑前数据收集；`no_agent=True` 让脚本成为整个 job）、**`context_from`**（把 job A 输出链入 job B）、`workdir`（在指定目录跑，加载其 `AGENTS.md`）。
- **不变式**：每次运行 3 分钟硬中断、`.tick.lock` 防跨进程重复 tick、cron 会话默认 `skip_memory=True`、投递用 header/footer 包裹而非镜像进目标会话（保角色交替）。

**对 Legion 的意义**：`context_from` 链式与 `skip_memory` 默认关都是 token 意识的设计；Legion 的 `taskledger` / `workflow` 若做定时链可直接借鉴。

<!-- @end-section -->

<!-- @section: session-search -->
## 7. session_search — FTS5 零成本历史检索

`05` 描述了 SessionDB（SQLite + FTS5）。v0.17 把它暴露为一等工具 `session_search`：

- **无 aux-LLM，几乎零成本**（纯 FTS5 检索，不消耗推理 token）。
- **一个工具三种模式**（按传入参数推断）：discovery（`query`）、scroll（`session_id` + `around_message_id`）、browse（无参数）。

**对 Legion 的意义**：★★★★★。这是**用检索替代历史堆叠**的核心手段——与其把长历史全塞进上下文，不如按需 FTS 检索相关片段。Legion **已实现**此能力（本计划 Task 4：FTS5 `conversation_turns_fts` + `session_search` 三模式工具，见 §token）。

<!-- @end-section -->

<!-- @section: memory -->
## 8. 记忆增强

`02` 已覆盖 MemoryProvider ABC + 8 种外部后端。v0.17 增量：

- **`memory` 工具批量操作**：传 `operations` 数组（add/replace/remove）**原子地**针对最终字符预算施加——单次调用即可「先腾空间再新增」，即使单独 add 会溢出预算。
- **新后端接入**：Supermemory 等持续增加；`hermes memory setup/status/off` 管理。
- **user_profile**：`memory.user_profile_enabled` 独立开关。

**对 Legion 的意义**：批量操作 + 字符预算是**记忆写入的 token 治理**范式，避免记忆无限膨胀撑爆上下文。Legion **已实现**（本计划 Task 3：`WorkingMemory.Apply` 原子 add/replace/remove，仅按最终字符预算校验，越界不改任何状态）。

<!-- @end-section -->

<!-- @section: ops -->
## 9. Profiles / Automation Blueprints / Relay / 新增交互面

- **Profiles**：`hermes profile <verb>`——多个独立实例，各自隔离 config/sessions/skills/memory，支持 clone/export/import。
- **Automation Blueprints**：命名自动化，Hermes 按需索要参数（无需 cron 语法）；一份定义同时渲染成 dashboard 表单、斜杠命令、对话、docs 目录条目。
- **Relay 多平台**：单 agent 对多平台（list identity / provision-loop / N-hello / per-frame egress）。
- **Gateway drain / accept-gating**：外部 drain 触发 + 接受门控（begin/cancel + control channel），优雅停机。
- **新增交互面**：`hermes proxy`（OpenAI 兼容本地代理，走 OAuth provider，无需 API key）、`hermes acp`（IDE 集成，VS Code/Zed/JetBrains）、`hermes desktop`（Electron，富渲染 embeds/图表/告警、子代理 watch-window）。

<!-- @end-section -->

<!-- @section: security -->
## 10. 安全 / 审批增强

- **smart 审批**：`approvals.mode` 三档——`manual`（默认，危险命令必问）、`smart`（aux-LLM 自动放行低风险、高风险才问）、`off`（等价 `--yolo`）。
- **Secret 脱敏**：默认开，import 期快照，**运行中不可切换**（防 LLM 自行关掉）。
- **PII 脱敏**：`privacy.redact_pii`——网关侧 hash 用户 ID、剥离电话号（独立于 secret 脱敏）。
- **tirith 熔断**：`fix(security): add circuit breaker for tirith crashes` 防 agent 挂起。

**对 Legion 的意义**：`smart` 审批的 aux-LLM 风险分级，与 Legion `approval` 模块可对接；secret 脱敏「快照式不可运行时切换」是防注入的重要不变式。

<!-- @end-section -->

<!-- @section: legion-token -->
## 11. 对 Legion 的新增参考：Token 消耗视角

将本轮新功能按「是否降低 token 消耗」重新归类，供 [[06-hermes-insights\|06]] 与后续 Legion 优化设计参考：

| Hermes 机制 | token 收益 | Legion 现状 |
|-------------|-----------|------------|
| session_search（FTS5，零 aux-LLM） | 检索替代历史堆叠，省重复 input token | **已实现**（本计划 Task 4：FTS5 `conversation_turns_fts` + `session_search` 三模式工具） |
| 子代理独立上下文（delegation） | 子任务中间过程不进父上下文 | **已实现**（本计划 Task 5：`delegate_task` batch/background + leaf/orchestrator 深度限制） |
| worker 专属工具集门控（kanban） | 按角色最小化工具 schema | LazyTools 已部分覆盖 |
| prompt 缓存不变式（中期不改 prompt/tools） | 复用 Anthropic 缓存，省全量 input | **已实现**（本计划 Task 2：`InferenceRequest.StablePrefixLen` + adapter `cache_control` 断点，`prompt_cache` 开关） |
| memory 批量操作 + 字符预算 | 记忆写入受控，不膨胀 | **已实现**（本计划 Task 3：`WorkingMemory.Apply` 原子批量对齐最终字符预算） |
| Curator 零成本确定性扫描 | 维护不烧 token | **已实现**（本计划 Task 7：`skill.Curator.Sweep` 确定性零 token 扫描 + stale/archive） |
| cron `skip_memory` 默认关 | 定时任务不背记忆开销 | N/A |

> 另：本计划 Task 1 补齐 CJK 感知 token 计数器（`cognitive.CJKTokenCounter`），是上述压缩/预算判定的准确性地基。MoA（Task 6：`runtime.MoACoordinator`）作为 one-shot 多模型协作能力落地。详细优化设计见 [[legion-hermes-optimization|Legion 优化计划]]（`docs/plans/`）。

<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[index|Hermes Agent 分析文档索引]]
- [[02-agent-runtime|Agent 运行时引擎分析]]（MoA / Goal / 中途注入 的基线）
- [[03-tools-skills-plugins|工具、技能与插件系统分析]]（Curator 的基线）
- [[04-gateway-cli-deployment|网关、CLI 与部署分析]]（Cron / Relay / 交互面 的基线）
- [[05-data-models|状态持久化与数据模型分析]]（session_search / 脱敏 的基线）
- [[06-hermes-insights|Hermes 洞察与 Legion 参考]]（差异化方向的延续）

<!-- @end-section -->
