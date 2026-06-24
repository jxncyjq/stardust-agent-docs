---
id: "analysis-mission-control-insights-007"
title: "Mission Control 设计洞察与 Legion 参考"
aliases: ["MC insights", "MC Legion参考", "MC设计洞察"]
type: "analysis"
category: "design/analysis/mission-control"
tags: ["mission-control", "legion", "design-reference", "lessons-learned", "orchestration"]
version: "1.0.0"
created: "2026-05-08"
updated: "2026-05-08"
author: "jxncyjq"
status: "review"
parent: "analysis-mission-control-index"
related_docs:
  - id: "analysis-mission-control-index"
    relation: "parent"
    path: "./index.md"
  - id: "analysis-mission-control-security-eval-005"
    relation: "related_to"
    path: "./05-security-eval-framework.md"
  - id: "analysis-mission-control-orchestration-003"
    relation: "related_to"
    path: "./03-orchestration-scheduler.md"
---

# Mission Control 设计洞察与 Legion 参考

<!-- @section: overview -->

## 文档目的

本文档从 Mission Control 的分析中提炼对 **Legion AI Agent 引擎**的设计参考，包括可直接复用的设计模式、值得借鉴的工程实践，以及 Legion 需要超越的方向。

## Mission Control 的核心价值定位

MC 本质上是一个 **AI Agent 舰队的指挥控制中心**，其核心能力矩阵：

| 能力 | 成熟度 | 对 Legion 的参考价值 |
|------|--------|---------------------|
| 任务状态机设计 | ★★★★★ | 直接参考六状态 + Aegis 门控 |
| 七种编排模式 | ★★★★★ | 参考完整编排模式分类 |
| Trust Score 信任评分 | ★★★★☆ | 参考加权事件 + 实时重算 |
| 四层评估框架 | ★★★★★ | 参考分层 Eval 设计 |
| FTS5 知识检索 | ★★★★☆ | 参考 BM25 + WikiLink 图谱 |
| 技能安全扫描 | ★★★★★ | 参考 13 条规则 + Critical/Warning 分级 |
| SOUL 人设系统 | ★★★★☆ | 参考双层持久化策略 |
| 调度器架构 | ★★★★☆ | 参考 13 任务 + 防重入设计 |
| 多租户架构 | ★★★☆☆ | 参考 Tenant→Workspace→User 层级 |
| 执行审批工作流 | ★★★★☆ | 参考人机协作门控 + Allowlist |

<!-- @end-section -->

<!-- @section: quality-gate -->

## 1. Aegis 质量门控 — 最重要的设计参考

### 精髓：任务不能绕过质量审核进入 done

MC 的最关键设计约束：**任务直接 PUT status=done 会返回 403**，必须经过 Aegis 审核。这不是一个可选特性，而是硬编码的数据库约束 + API 层检查双重保障。

```
任务完成路径（必须）：
in_progress → quality_review → [Aegis 审核] → done

任何绕过路径：
in_progress → done 直接写入  ← 403 FORBIDDEN
```

**三轮驳回上限**：Aegis 最多驳回 3 次，超过则任务自动转为 `failed`，防止无限循环。

### Legion 可复用

```
Legion Agent 引擎的质量门控设计：
1. 每个输出任务挂接 evaluator Agent（可配置是否强制）
2. 强制质量关的任务：done 写入需要 eval_runs.result = 'approved'
3. evaluator 反馈循环：最多 N 轮（默认 3），超出转 failed
4. 轻量模式：可配置 skip_review_for = ['routine', 'low_priority']
```

<!-- @end-section -->

<!-- @section: trust-score -->

## 2. 动态信任评分 — 行为可观测的基础

### 精髓：事件驱动 + 实时重算 + 工作区隔离

MC 的信任评分不是定时任务，而是**每次安全事件发生时实时重算**。权重设计体现了风险优先级：

```
注入攻击尝试（-0.15）> 密钥泄露（-0.20）> 认证失败（-0.05）
任务成功（+0.02）— 正向激励，但恢复很慢（50 次成功 = 1 次密钥泄露）
```

**调制效应**：信任分不仅用于单 Agent 评估，还调制工作区整体安全态势评分（乘法关系），形成"舰队可信度 → 整体风险"的传导链。

### Legion 可复用

```rust
// Legion 信任评分设计
pub struct AgentTrustProfile {
    score: f64,                          // 0.0 ~ 1.0
    event_counters: HashMap<EventType, u32>,
    last_anomaly_at: Option<DateTime<Utc>>,
    workspace_id: WorkspaceId,
}

impl AgentTrustProfile {
    fn recalculate(&mut self, event: &SecurityEvent) {
        // 事件发生时实时重算，不用定时任务
        let delta = TRUST_WEIGHTS.get(&event.event_type)
            .map(|w| w.delta)
            .unwrap_or(0.0);
        self.score = (self.score + delta).clamp(0.0, 1.0);
        if delta < 0.0 {
            self.last_anomaly_at = Some(Utc::now());
        }
    }
}
```

**Legion 扩展建议**：
- 按任务类型细粒度追踪成功/失败（而非仅 task.success/failure）
- 引入信任分恢复期冷却机制（异常事件后 N 小时内分数不回升）
- 跨工作区信任记录（Agent 可能在多个工作区间迁移）

<!-- @end-section -->

<!-- @section: four-layer-eval -->

## 3. 四层评估框架 — 系统化评估体系

### 精髓：递进式、多维度、可量化

```
Layer 1 Output  → "任务完成了吗？"         （7天维度，≥70% pass）
Layer 2 Trace   → "Agent 在循环吗？"       （24h，工具重复率≤3.0x）
Layer 3 Component → "工具可靠吗？"         （成功率≥80%）
Layer 4 Drift   → "行为漂移了吗？"         （10% 相对变化阈值）
```

**收敛性指标的创意**：`ratio = totalToolCalls / uniqueTools`，ratio > 3.0 判定为循环。这个指标简单有效：正常 Agent 应该使用多样化工具，反复调用相同工具是卡死的信号。

### Legion 可复用

```rust
// 收敛性评估（直接可用）
pub fn convergence_score(trace: &ExecutionTrace) -> f64 {
    let ratio = trace.total_tool_calls as f64 / trace.unique_tools as f64;
    f64::min(1.0, 3.0 / ratio)
}

// 漂移检测模式（可泛化到任意指标）
pub fn detect_drift(current: f64, baseline: f64, threshold: f64) -> bool {
    (current - baseline).abs() / baseline > threshold
}
```

**Legion 扩展建议**：
- Layer 5：跨 Agent 协作质量评估（接力任务的整体成功率）
- Layer 6：成本效率评估（每单位输出的 token 消耗 vs 同类 Agent）
- 结合 Legion 的 GEP 进化引擎：评估结果驱动 Agent 技能自动更新

<!-- @end-section -->

<!-- @section: scheduler -->

## 4. 调度器设计 — 简单但有效的后台任务系统

### 精髓：防重入 + 动态启用 + 分散初始延迟

MC 调度器的工程细节值得关注：

**防重入**：`task.running = true` flag 防止调度任务相互叠加（任务执行慢于间隔时不会重复触发）。

**分散初始延迟**：12 个任务的首次延迟各不相同（5s/10s/15s/20s/25s/30s/60s），防止服务启动时所有任务同时触发，引发 SQLite 写入争用。

**动态启用**：每次 tick 都从 `settings` 表重读任务开关，运维人员可在不重启服务的情况下禁用/启用特定调度任务。

### Legion 可复用

```rust
pub struct ScheduledTask {
    id: &'static str,
    interval: Duration,
    initial_delay: Duration,
    enabled_key: &'static str,          // settings 表的 key
    running: AtomicBool,                // 防重入
    next_run: AtomicI64,
}

impl ScheduledTask {
    async fn tick(&self, db: &Db) {
        if self.running.swap(true, Ordering::SeqCst) {
            return;  // 已在运行，跳过
        }
        // 执行 ...
        self.running.store(false, Ordering::SeqCst);
    }
}
```

**Legion 差异**：Legion 的调度器还需支持**分布式锁**（多实例场景），MC 是单进程设计。

<!-- @end-section -->

<!-- @section: skill-security -->

## 5. 技能安全扫描 — 防御性技能生态

### 精髓：安装时静态扫描 + Critical/Warning 分级响应

MC 在技能安装前执行 13 条安全规则扫描，分两级响应：
- **Critical**：阻断安装（保护系统）
- **Warning**：允许安装但标记（保留可见性）

这种设计的巧妙之处在于**不是全拒绝，而是分级透明**。Warning 级的技能可以安装，但 `security_status='warning'` 字段让管理员知晓风险。

**关键防御点**：
1. 提示注入防御（"ignore previous instructions"）
2. SSRF 防御（内网地址/云元数据端点）
3. 数据外泄防御
4. 路径穿越防御
5. 混淆内容检测（base64/hex 编码）

### Legion 可复用

Legion 的技能/工具安全扫描应参考此设计，特别是：
- 对接 Legion 工具注册表时，安装前强制扫描
- 扫描规则版本化（规则库可升级，不影响已安装技能）
- 扫描结果写入不可篡改日志（Ed25519 签名，参考 MC 的 mcp_call_log）

<!-- @end-section -->

<!-- @section: soul-dispatch -->

## 6. SOUL + Auto-Dispatch 的人设注入机制

### 精髓：人设与派遣解耦，运行时动态注入

MC 将 Agent 人设（SOUL.md）与任务内容（task description）分离，在 auto-dispatch 时动态拼接：

```
dispatch_prompt = SOUL.md + "\n\n" + task_description + "\n\n" + project_context
```

这种设计的优势：
1. 人设可以独立更新，不影响已有任务
2. 同一 Agent 接受不同任务时，人设始终一致
3. 人设可以被模板化（多个 Agent 共享同一基础人设）

**双层持久化**：workspace 文件优先于 DB，运维时直接编辑 soul.md 文件即可更新，无需通过 API。

### Legion 可复用

Legion 的 Cognitive Core 公式：

```
执行上下文 = 角色人设 + 任务指令 + 目标传导链 + 经验记忆 + 挂载技能 + 组织上下文 + 约束规则
```

MC 的 SOUL 对应"角色人设"部分。Legion 可扩展为：
- SOUL.md（静态基础人设）+ 动态技能挂载（per-task）+ 组织上下文注入（org hierarchy）

<!-- @end-section -->

<!-- @section: architecture-comparison -->

## 7. MC 架构 vs Legion 架构的差异分析

### 7.1 相同的核心理念

| 维度 | Mission Control | Legion |
|------|----------------|--------|
| Agent 人设 | SOUL.md | 角色人设（Legion.md §3）|
| 质量门控 | Aegis 审核 | 评估门控（7 governance gates）|
| 信任评分 | Trust Score | 可信度评估（Legion.md §5.3）|
| 编排模式 | 7 种编排 | Workflow DSL（8 原语）|
| 多租户 | Tenant→Workspace→User | Company→Department→Team→Agent |

### 7.2 Legion 需要超越的方向

**超越 1：多层级组织建模**

MC 只有 Tenant→Workspace 两级。Legion 需要 Company→Department→Team→Agent 四级，支持跨团队任务协作、向上的目标传导链。

**超越 2：基于语义的任务路由**

MC 的路由是基于关键词匹配（"debug" → Opus）和角色匹配（`agent.role`）。Legion 应支持：
- 向量相似度匹配（任务描述 embedding vs Agent 能力 embedding）
- 历史成功率权重（哪个 Agent 最擅长此类任务）
- 实时负载感知（选择当前最空闲且合适的 Agent）

**超越 3：自进化能力**

MC 只有"评估"没有"进化"。Legion 的 GEP 引擎应基于评估结果驱动：
- 技能自动推荐（低评估分 → 推荐相关技能）
- 提示词自动优化（任务失败 → 分析 → 生成更好的提示模板）
- Agent 能力图谱动态更新

**超越 4：成本感知模型路由**

MC 的模型路由是规则式（critical→Opus）。Legion §1.8 的 10 步路由算法更精细：
- 角色默认层级 + 任务复杂度评分 + 预算压力 + 能力需求 + 延迟 SLA
- 动态成本调制（月末超预算 → 自动降级路由）

**超越 5：跨系统 MCP 审计链**

MC 的 mcp_call_log 已有 Ed25519 签名，但仅记录 MC 本身的调用。Legion 需要：
- 跨 Agent 工具调用的完整追踪链（调用链 propagation）
- Runs 表的 lineage 字段形成有向无环图（DAG）
- 支持任意节点回溯到根 trigger

<!-- @end-section -->

<!-- @section: pitfalls -->

## 8. 坑点与教训

### 坑点 1：任务状态残留问题

MC 中 `review` 状态在 Migration 003 被废弃并迁移到 `quality_review`，但代码中仍有残留的 `review` 字符串判断。**Legion 建议**：状态枚举在 Rust 中定义为 `enum`，编译器强制穷举匹配，彻底消除"废弃状态字符串残留"问题。

### 坑点 2：两套定时系统职责混淆

MC 有两套定时系统：
1. Node.js 调度器（内部运维：同步、备份、任务分发）
2. OpenClaw Cron（业务层面：定时向 Agent 发消息）

这两套系统的职责划分不够清晰，UI 中混合展示容易让用户困惑。**Legion 建议**：明确区分"系统调度"（对用户不可见）和"业务 Cron"（对用户开放配置）。

### 坑点 3：assigned_to 字段是字符串而非 FK

`tasks.assigned_to` 存储的是 Agent 名称字符串，而非 `agents.id` 的外键。这导致：
- Agent 改名后所有历史任务的 assigned_to 就变成悬空引用
- 无法利用 DB 外键约束保证一致性

**Legion 建议**：任务-Agent 关联应使用稳定的内部 ID（UUID），name 只用于显示。

### 坑点 4：MCP 工具数量膨胀

MC 的 MCP Server 有 35 个工具，其中部分功能重叠（如 `mc_agent_costs` 和 `mc_costs_by_agent`，`mc_list_sessions` 和 `mc_control_session` 等）。**Legion 建议**：工具设计遵循"正交原则"——每个工具做一件事，通过参数组合覆盖变体，而非为每个变体创建新工具。

### 坑点 5：SQLite WAL 在高并发下的局限

MC 是单进程架构，SQLite WAL 完全足够。但 `task.status = 'in_progress'` 的原子 UPDATE-RETURNING 依赖 SQLite 单写锁，在高并发（多个 Agent 同时抢任务）场景下会有队列等待。**Legion 建议**：大规模部署时需要数据库层面的任务分发改为 Redis 队列 + 乐观锁。

<!-- @end-section -->

<!-- @section: direct-reuse -->

## 9. 可直接复用的设计模式

### 立即可用

| 模式 | 来源 | Legion 应用位置 |
|------|------|----------------|
| 任务六状态机 + Aegis 门控 | `schema.sql` + `api/tasks` | Legion 任务执行引擎 |
| Trust Score 加权事件实时重算 | `lib/security-events.ts` | Legion 安全评估模块 |
| 四层 Eval 框架（Output/Trace/Component/Drift）| `lib/agent-evals.ts` | Legion 评估引擎 |
| SOUL 双层持久化（disk > DB）| `api/agents/[id]/soul` | Legion Agent 人设系统 |
| FTS5 BM25 + WikiLink 图谱 | `lib/memory-search.ts` | Legion LLM Wiki |
| 技能 13 条安全扫描规则 | `lib/skill-registry.ts` | Legion 技能市场 |
| 执行审批 Allowlist + CAS 并发控制 | `api/exec-approvals` | Legion 工具审批门 |
| Webhook 退避表 + 熔断器 | `lib/webhooks.ts` | Legion 事件推送 |
| Ed25519 MCP 调用签名 | `migrations.ts` M050 | Legion 审计链 |
| 调度器防重入 + 分散初始延迟 | `lib/scheduler.ts` | Legion 后台任务引擎 |

### 改进后使用

| 方面 | MC 现状 | Legion 改进建议 |
|------|---------|----------------|
| 任务路由 | 关键词匹配 + 角色匹配 | 语义向量匹配 + 历史成功率权重 |
| 模型路由 | 2 级规则（Opus/Haiku/默认）| 10 步路由算法（见 agent-engine-design §1.8）|
| 编排模式 | 7 种定性模式 | 8 原语 Workflow DSL（可形式化组合）|
| 信任评分恢复 | 无冷却期（成功即加分）| 引入冷却窗口，防止刷分 |
| 组织层级 | Tenant→Workspace（2 级）| Company→Department→Team→Agent（4 级）|
| 技能进化 | 无（安装 + 扫描即止）| 评估结果驱动技能自动推荐/更新 |

<!-- @end-section -->

<!-- @section: related -->

## 相关文档

- [[index|分析索引]]
- [[01-overview|01 项目总览]]
- [[03-orchestration-scheduler|03 编排引擎与调度器]]
- [[05-security-eval-framework|05 安全框架与 Agent 评估]]
- [[../../architecture/agent-engine-design|Legion Agent 引擎设计方案]]
- [[../../architecture/Legion|Legion V3.0 项目方案]]

<!-- @end-section -->
