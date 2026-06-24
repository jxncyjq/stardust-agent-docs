---
id: "analysis-clawcode-gap-007"
title: "Claw Code 差距分析与待完成工作"
aliases: ["差距分析", "gap analysis", "claw-code 待办", "技术债务"]
type: "analysis"
category: "design/analysis/claude"
tags: ["claw-code", "gap-analysis", "tech-debt", "roadmap", "P0", "P1"]
version: "1.0.0"
created: "2026-05-03"
updated: "2026-05-03"
author: "jxncyjq"
status: "review"
parent: "analysis-clawcode-overview-001"
children: []
related_docs:
  - id: "analysis-clawcode-overview-001"
    relation: "depends_on"
    path: "./01-overview.md"
  - id: "analysis-clawcode-checklist-004"
    relation: "references"
    path: "./04-requirements-checklist.md"
---

<!-- @section: overview -->
# Claw Code 差距分析与待完成工作

> 基于 `claw-code/ROADMAP.md`、`claw-code/PARITY.md`、`claw-code/progress.txt`、`claw-code/prd.json` 的全面差距分析。文中引用的 commit 哈希按当时仓库快照记录，本仓库工作副本未保留 git 历史，需要在上游 `ultraworkers/claw-code` 仓库中核验。

---

## 一、已完成里程碑总览

### PRD 全部交付（24/24 用户故事）

| 阶段 | 故事数 | 范围 |
|------|--------|------|
| 阶段 1 (Worker Boot) | 1 | 启动超时证据包 + 分类器 |
| 阶段 2 (Lane Events) | 9 | 规范 lane 事件 schema、排序、溯源、去重、身份、所有权 |
| 阶段 3 (Branch/Recovery) | 2 | 过期分支检测、恢复配方 + 账本 |
| 阶段 4 (Task Execution) | 2 | 类型化任务包、自主编码策略引擎 |
| 阶段 5 (Plugin/MCP) | 1 | 插件/MCP 生命周期成熟度 |
| API 兼容性 | 9 | kimi-k2.5 兼容、测试、文档、性能、信任解析器、请求体检查、错误上下文、kimi 路由、令牌限制 |
| **合计** | **24** | **全部 passes: true** |

### 9 通道检查点（全部已合并 `main`）

| # | 通道 | 功能提交 | 合并提交 |
|---|------|----------|----------|
| 1 | Bash 验证 | `36dac6c` | `1cfd78a` |
| 2 | CI 修复 | `89104eb` | `f1969ce` |
| 3 | File-tool 边界 | `284163b` | `a98f2b6` |
| 4 | TaskRegistry | `5ea138e` | `21a1e1d` |
| 5 | Task wiring | `e8692e4` | `d994be6` |
| 6 | Team+Cron | `c486ca6` | `49653fe` |
| 7 | MCP lifecycle | `730667f` | `cc0f92e` |
| 8 | LSP client | `2d66503` | `d7f0dc6` |
| 9 | Permission enforcement | `66283f4` | `336f820` |

### 即时待办清单 P0/P1/P2（全部核心项已完成）

30+ 项，包括：Worker 就绪握手、信任解决、跨模块集成测试、通道完成发射器、摘要压缩集成、提示误投递检测、失败分类法、MCP 降级启动报告、任务包结构、配置合并验证、上下文窗口预检、session 状态分类、doctor JSON 结构、插件生命周期 flake 修复、钩子 broken-pipe 竞态修复。

---

## 二、当前进行中的工作

### Ship 事件连接（§4.44.5.1 — 实现但未连接）

- `lane_events.rs` 中的 `ShipProvenance` 和 `LaneEvent::ship_*()` **已实现**
- 但 Git 推送操作（main.rs / tools/lib.rs / worker_boot.rs）**不发出**这些事件
- 需要连接：`ship.prepared` → `ship.commits_selected` → `ship.merged` → `ship.pushed_main`

### Typed-error envelope contract（§4.44 — 设计已完成，未实现）

将 6 个不同 pinpoint（#102, #121, #127, #129, #130, #245）合并为单个合同：
- 结构化 `error.kind` 枚举 (filesystem/auth/session/parse/runtime/mcp/delivery/usage/policy/unknown)
- `error.operation` / `error.target` / `error.errno` / `error.hint` / `error.retryable`
- 人类文本 vs JSON 双渲染
- 向后兼容保证

---

## 三、计划中但未开始的功能

### A. 报告合同（路线图阶段 2：§4.13–§4.43，30 项，全部 P2）

| 章节 | 项目 |
|------|------|
| §4.13 | 多消息报告原子性 |
| §4.14 | 跨 Claw pinpoint 去重/合并 |
| §4.15 | Pinpoint 证据附件合同 |
| §4.16 | Pinpoint 优先级/严重性合同 |
| §4.17 | Pinpoint 到实施的交接合同 |
| §4.18 | 报告反压/重复摘要折叠 |
| §4.19 | 无变更/无操作确认 |
| §4.20 | 观察新鲜度/陈旧年龄 |
| §4.21 | 事实/假设/信心标签 |
| §4.22 | 负面证据/已搜索未找到合同 |
| §4.23 | 字段级增量归因 |
| §4.24 | 报告 schema 版本化 |
| §4.25 | 消费者能力协商 |
| §4.26–4.34 | 自描述/投影/脱敏/确定性/夹具/一致性测试 |
| §4.35–4.40 | 临时状态、策略阻止、批准令牌治理 |
| §4.41–4.43 | 令牌优化/工作区权重预览/安全范围快速应用 |

### B. 绿色级别合同/活跃度（路线图阶段 3：§9 / §12.5）

- 目标/test/package/workspace 四级绿色级别 + 测试超时强制执行
- `test.hung` 分类
- 运行状态活跃度心跳（阶段/进度秒数/活动步骤/后台预期）

### C. 其他指定但未连接的领域

- **§5.5** 传输中断 vs 通道失败边界
- **§6** 可操作摘要压缩（阶段/检查点/阻塞器/恢复操作）
- **§14** MCP 端到端生命周期对等（配置→注册→连接→握手→发现→调用→关闭）

---

## 四、已知技术债务

### CI / 测试（5 项）

| ID | 描述 | 严重性 |
|----|------|--------|
| #140 | `permissionMode` 迁移降级导致 1 个测试失败，`cargo test --workspace` 为红色 | P0 |
| #137 | feat/134-135 分支上模型别名回归（3 个测试失败） | P0 |
| #149 | `runtime::config` 并行测试 flake | P1 |
| #150 | `resume_latest` 符号链接不匹配 flake | P1 |
| PARITY | 每次提交 CI 绿色未达成 | P0 |

### 架构 / 运行时（7 项）

| ID | 描述 | 严重性 |
|----|------|--------|
| #160 | `session_store` 缺少 `list_sessions`/`delete_session`/`session_exists` | P1 |
| #158 | `compact_messages_if_needed` 静默丢弃轮次，无结构化压缩事件 | P2 |
| #159 | `run_turn_loop` 硬编码空 denied_tools，静默缺失权限拒绝 | P2 |
| #133 | Blocked-state 子阶段合同：`lane.blocked` 是单个不透明状态 | P2 |
| §4.44.5.1 | Ship 事件已实现但为死代码（未连接 Git 推送调用点） | P1 |
| §4.44 | Typed-error envelope contract 未实现 | P0 |
| Bash | Bash 工具：上游 18 个子模块 vs Rust 1 个 | P2 |

### CLI Surface / 可发现性（12 项）

| ID | 描述 |
|----|------|
| #122 | `doctor` 不检查过期基础条件 |
| #134 | 会话边界无运行/关联 ID |
| #135 | `claw status --json` 缺少 `active_session` + `session.id` |
| #136 | `--compact` 标志在 JSON 模式输出纯文本 |
| #139 | `claw state` 错误消息引用不可发现的 "worker" 概念 |
| #141 | `claw <subcommand> --help` 有 5 种不同行为 |
| #142 | `claw init --output-format json` 无结构化 created/skipped 字段 |
| #143+144 | `claw status`/`claw mcp` 在格式错误的 MCP 配置上硬失败 |
| #145 | `claw plugins` 子命令未连接到 CLI 解析器 |
| #146 | `claw config`/`claw diff` 需要 `--resume` 包装器 |
| #147 | `claw ""` / `claw "   "` 静默回退到 prompt 执行 |
| #148 | `claw status` JSON 不显示原始模型输入或来源 |

### 文档（3 项）

| ID | 描述 |
|----|------|
| #153 | README/USAGE 缺少 PATH 和安装验证桥梁 |
| #154 | 模型语法错误不提示环境变量 |
| #155 | USAGE.md 缺少 `/ultraplan`、`/teleport`、`/bughunter` 文档 |

### 启动摩擦（2 项）

| ID | 描述 |
|----|------|
| #246 | 提醒 cron 结果模糊：无结构化交付/跳过/超时反馈 |
| #243 | 启动提示在未预览具体变更计划的情况下征求同意 |

---

## 五、与上游 TypeScript 的差距

| 差距项 | 详细描述 |
|--------|----------|
| **Bash 验证矩阵** | 上游 TypeScript 实现按 `PARITY.md` 记录有 18 个子模块（来源：上游 `bash/*` 目录列表，需到 `ultraworkers/claw-code` 上游 TypeScript 树核验），Rust 当前合并到 `bash_validation.rs` 单文件 6 类验证（readOnly/destructive/mode/sed/path/commandSemantics）。差距是验证矩阵的覆盖率，不是文件数 |
| **端到端 MCP 运行时** | 当前仅注册表/调度级。外部进程编排、传输深度和运行时集成仍然分离 |
| **LSP 工具 Surface** | completion/format 在注册表中存在，但工具 schema 边界未清晰暴露 |
| **AskUserQuestion** | 返回 pending 响应，无真正交互式 UI 连接 |
| **RemoteTrigger** | 存根响应 |
| **会话压缩行为** | 行为匹配尚未完成 |
| **Token 计数准确性** | 成本跟踪准确性未完全对等 |
| **MCP 配置互操作** | 格式错误的 MCP 配置在 status/mcp 硬失败 vs doctor 优雅降级，无迁移路径 |

---

## 六、按优先级排列的待办事项

### P0 — 立即阻塞

1. **修复 #140** — `permissionMode` 迁移降级，1 个确定性测试失败
2. **修复 #137** — feat/134-135 模型别名回归（合并前）
3. **实现 §4.44 Typed-error envelope contract** — 汇总块（#102, #121, #127, #129, #130, #245）
4. **PARITY: CI 绿色每个提交** — 当前 CI 非绿色

### P1 — 连接集成（实现但未连接 / 关键缺失）

5. **连接 Ship 事件** — `lane_events.rs` 中已实现但从未调用
6. **实现 #134** — 会话启动时运行/关联 ID
7. **实现 #135** — `claw status --json` 获取 `active_session` + `session_id`
8. **修复 #122** — `doctor` 调用站点缺失过期基础检查
9. **修复 #145** — `claw plugins` 子命令解析
10. **修复 #147** — 空提示保护
11. **修复 #160** — 添加 `list_sessions`/`delete_session`/`session_exists`

### P2 — 可抓取性强化

12. **修复 #141** — 统一 `claw <subcommand> --help` 行为
13. **修复 #143+144** — MCP 配置降级（不应硬失败）
14. **实现 #133** — Blocked-state 子阶段合同
15. **修复 #136** — `--compact` JSON 输出
16. **修复 #158+159** — 结构化压缩事件 + 权限拒绝保留
17. **修复 #142** — `claw init --json` 结构化字段

### P2 — 路线图阶段 2 报告合同（最高优先级候选）

18. **§4.13** — 多消息报告原子性
19. **§4.16** — Pinpoint 优先级/严重性合同
20. **§4.17** — Pinpoint 到实施的交接合同

### P3 — 文档 / 可发现性

21. **修复 #153** — README/USAGE PATH 桥梁
22. **修复 #155** — 高级斜杠命令文档
23. **修复 #139** — Worker 概念可发现性

### 远期路线图（§4.14–§4.43 剩余 25+ 项）

完整的报告生态系统合同：交叉 pinpoint 去重、证据附件、事实/假设标签、schema 版本化、消费者能力协商、脱敏、确定性投影、回归夹具、下游测试、批准令牌治理、工作区权重预览。

### PARITY "Still open" (未在路线图逐步分解)

- 会话压缩行为匹配
- Token 计数 / 成本跟踪准确性
- 每次提交 CI 绿色
<!-- @end-section -->

<!-- @section: related -->
## 相关文档

- [[01-overview|项目架构总览]]
- [[04-requirements-checklist|需求清单]]
<!-- @end-section -->

---

## 七、统计总结

| 分类 | 数目 |
|------|------|
| 已完成的 PRD 用户故事 | 24/24 |
| 已合并的通道 | 9/9 |
| 已完成的即时待办 | 30+ |
| 进行中的实时差距 | 2 (Ship 事件连接 + Typed-error) |
| 设计完成但未实现 | 30+ (报告合同等) |
| 已知技术债务项 | 29（CI/测试 5 + 架构/运行时 7 + CLI Surface 12 + 文档 3 + 启动摩擦 2） |
| **上游差距项** | 8 |
