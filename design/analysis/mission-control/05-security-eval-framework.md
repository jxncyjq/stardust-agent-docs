---
id: "analysis-mission-control-security-eval-005"
title: "Mission Control 安全框架与 Agent 评估"
aliases: ["MC security", "MC eval", "MC安全评估"]
type: "analysis"
category: "design/analysis/mission-control"
tags: ["mission-control", "security", "trust-score", "eval", "mcp-audit", "injection", "four-layer"]
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
  - id: "analysis-mission-control-memory-skills-004"
    relation: "related_to"
    path: "./04-memory-skills-hub.md"
---

# Mission Control 安全框架与 Agent 评估

<!-- @section: trust-score -->

## 1. Agent 信任评分系统

### 1.1 设计理念

每个 Agent 维护一个 `trust_score`（0~1 浮点数），从满分 1.0 开始，随各类安全事件逐步调整。信任分影响：
- 安全态势（Security Posture）评分调制
- 高风险任务的派遣决策
- 审计日志的风险标记

### 1.2 事件权重表

```typescript
const TRUST_WEIGHTS = {
  'auth.failure':       { field: 'auth_failures',      delta: -0.05 },
  'injection.attempt':  { field: 'injection_attempts', delta: -0.15 },
  'rate_limit.hit':     { field: 'rate_limit_hits',    delta: -0.03 },
  'secret.exposure':    { field: 'secret_exposures',   delta: -0.20 },  // 最重
  'task.success':       { field: 'successful_tasks',   delta: +0.02 },
  'task.failure':       { field: 'failed_tasks',       delta: -0.01 },
}
```

**计算公式**：

```
trust_score = clamp(
  1.0
  + auth_failures × (-0.05)
  + injection_attempts × (-0.15)
  + rate_limit_hits × (-0.03)
  + secret_exposures × (-0.20)
  + successful_tasks × 0.02
  + failed_tasks × (-0.01),
  0, 1
)
```

**触发时机**：每次 `logSecurityEvent()` 调用时实时重算，不使用批量定时更新。

### 1.3 数据库结构

```sql
CREATE TABLE agent_trust_scores (
  agent_name          TEXT NOT NULL,
  workspace_id        INTEGER NOT NULL,
  trust_score         REAL DEFAULT 1.0,
  auth_failures       INTEGER DEFAULT 0,
  injection_attempts  INTEGER DEFAULT 0,
  rate_limit_hits     INTEGER DEFAULT 0,
  secret_exposures    INTEGER DEFAULT 0,
  successful_tasks    INTEGER DEFAULT 0,
  failed_tasks        INTEGER DEFAULT 0,
  last_anomaly_at     INTEGER,    -- 最后一次负向事件时间
  updated_at          INTEGER DEFAULT (unixepoch()),
  PRIMARY KEY (agent_name, workspace_id)
)
```

**幂等插入**：`INSERT OR IGNORE` 保证 Agent 首次出现时自动创建记录。

<!-- @end-section -->

<!-- @section: security-posture -->

## 2. 安全态势评估

### 2.1 工作区级安全态势

`getSecurityPosture(workspaceId)` 返回整体安全评分（0~100）：

```
┌─────────────────────────────────────────────────────┐
│  score = 100                                         │
│        - criticalEvents × 10                        │
│        - warningEvents × 3                          │
│        - recentIncidents × 2                        │  ← 最近24小时
│  score = round(score × avgTrustScore)               │  ← 信任分调制
│  score = clamp(score, 0, 100)                       │
└─────────────────────────────────────────────────────┘
```

**调制效果**：若所有 Agent 信任分均值为 0.7，则基础安全分再乘 0.7，体现舰队整体可信度对安全态势的影响。

### 2.2 安全事件

**事件级别**：`info` / `warning` / `critical`

**事件类型**（不限于以下）：
- `auth.failure`：认证失败
- `injection.attempt`：注入攻击尝试
- `rate_limit.hit`：触发速率限制
- `secret.exposure`：敏感信息泄露
- `api.key.usage`：全局 API Key 使用（推荐改用 agent-scoped key）
- `proxy.auth.untrusted_ip`：代理认证来自不可信 IP（critical）

**广播机制**：每个安全事件写入 `security_events` 表后，同步通过 `eventBus.broadcast('security.event')` 广播，UI 实时更新。

<!-- @end-section -->

<!-- @section: auth-system -->

## 3. 认证系统

### 3.1 五层认证优先级

```
优先级 1：Proxy Auth（代理认证）
  MC_PROXY_AUTH_HEADER + MC_PROXY_AUTH_TRUSTED_IPS
  适用于 Envoy/OIDC 网关前置场景
  不可信 IP 使用该 Header → security_events（critical）

优先级 2：Session Cookie（mc-session）
  32 字节随机 token，SHA-256 哈希存入 DB
  有效期 7 天，每次登录清理过期 session
  Migration 043 对存量 token 全量哈希化

优先级 3：全局 API Key（X-Api-Key）
  读取 settings.security.api_key 或 API_KEY 环境变量
  timingSafeEqual 防时序攻击
  每次使用记录安全事件（info 级别，提示改用 agent-scoped key）
  合成用户：{id:0, username:'api', role:'admin'}

优先级 4：Agent 专属 API Key（agent_api_keys）
  key_hash = SHA-256(rawKey)
  scopes JSON 数组决定 role（admin/operator/viewer）
  支持 expires_at / revoked_at
  X-Agent-Name Header 必须与 key 绑定的 agent 匹配

优先级 5：插件钩子（registerAuthResolver）
  允许扩展注入自定义认证逻辑
```

### 3.2 RBAC 模型

```
admin > operator > viewer

ROLE_LEVELS = { viewer: 0, operator: 1, admin: 2 }
```

| 角色 | 代表权限 |
|------|---------|
| viewer | GET 查询、心跳 GET |
| operator | 创建任务/Agent、更新状态、心跳 POST、记忆写入 |
| admin | 用户管理、cron 管理、安全配置、删除操作 |

### 3.3 密码安全

**哈希算法**：`scrypt`（原本支持 progressive rehash — 登录时若使用旧 cost 参数自动升级）

**计时攻击防护**：`DUMMY_HASH` 恒等比较防止 username 枚举计时攻击

**密码强度**：最小 12 字符

### 3.4 Google OAuth 流程

```
用户 Google 登录回调
  │
  ├─ 首次登录且 is_approved = false
  │    └─ 写入 access_requests（attempt_count 去重幂等）
  │    └─ 返回"等待审批"
  │
  └─ is_approved = true
       └─ 创建 session，正常登录
       └─ approved_by / approved_at 记录审批人

管理员操作：
  PATCH /api/users/{id} → is_approved = true, approved_by = adminName
```

### 3.5 Session Token 安全升级历史

Migration 043（重要安全更新）：将 `user_sessions.token` 从明文迁移到 SHA-256 哈希存储，存量 session 全部失效（用户需要重新登录）。这是典型的"先存哈希"设计，即使 DB 泄露，攻击者也无法直接使用 token。

<!-- @end-section -->

<!-- @section: four-layer-eval -->

## 4. 四层 Agent 评估框架

### 4.1 框架概览

```
Layer 1 Output     ← 任务完成了吗？用户满意吗？
    │
    ▼
Layer 2 Trace      ← Agent 行为模式是否合理？
    │
    ▼
Layer 3 Component  ← 工具调用是否可靠？
    │
    ▼
Layer 4 Drift      ← 行为是否发生漂移？
```

### 4.2 Layer 1 - Output Eval（产出评估）

**任务完成率**：
```
时间窗口：最近 168 小时（7 天）
指标：done / (done + failed + in_progress)
阈值：≥ 70% pass
```

**正确率评分**：
```
correctness = successRate × 0.6 + normalizedRating × 0.4

normalizedRating = (feedback_rating - 1) / 4  （1~5 分归一化为 0~1）
无反馈时：correctness = successRate × 1.0

阈值：≥ 60% pass
```

### 4.3 Layer 2 - Trace Eval（追踪评估）

**收敛性分析**（检测循环行为）：

```
数据源：mcp_call_log（最近 24 小时）

convergenceScore = min(1.0, 3.0 / (totalToolCalls / uniqueTools))

ratio = totalToolCalls / uniqueTools
  ratio > 3.0  → LOOPING（工具重复率过高，Agent 陷入循环）
  ratio ≤ 3.0  → CONVERGING（正常收敛）
```

**示例**：
- 10次调用 × 8 种不同工具 → ratio=1.25 → convergenceScore=1.0（良好）
- 30次调用 × 3 种不同工具 → ratio=10.0 → convergenceScore=0.3（存在循环风险）

### 4.4 Layer 3 - Component Eval（组件评估）

**工具可靠性**：
```
数据源：mcp_call_log
指标：成功调用次数 / 总调用次数
阈值：≥ 80% pass
```

**按工具细粒度分析**：每种工具单独统计成功率，识别哪个工具存在稳定性问题。

### 4.5 Layer 4 - Drift Detection（漂移检测）

**检测对比**：当前周（最近 7 天）vs 基线（4 周前~1 周前的 3 周均值）

**三个检测指标**：

| 指标 | 数据源 | 漂移阈值 |
|------|--------|---------|
| `avg_tokens_per_session` | token_usage 表 | 相对变化 > 10% |
| `tool_success_rate` | mcp_call_log | 相对变化 > 10% |
| `task_completion_rate` | tasks 表 | 相对变化 > 10% |

**8 周历史 Timeline**：`GET /api/evals/{agentId}/drift?weeks=8` 提供历史趋势数据。

**漂移判定**：
```
drift = abs(current - baseline) / baseline > 0.10
```

### 4.6 优化建议（Optimize API）

在评估基础上，`GET /api/evals/{agentId}/optimize` 提供：
- Token 效率分析（每任务平均 token 消耗趋势）
- 工具调用模式分析（调用分布、高频工具识别）
- 舰队百分位排名（与同类 Agent 横向对比）
- 具体优化建议列表

### 4.7 Golden Set 管理

`eval_golden_sets` 表维护预期行为黄金数据集：

```sql
CREATE TABLE eval_golden_sets (
  id          INTEGER PRIMARY KEY,
  agent_id    INTEGER,
  input       TEXT,   -- 输入（任务描述）
  expected    TEXT,   -- 预期输出
  category    TEXT,   -- 类别（如 code_review, security_audit）
  created_by  TEXT,
  workspace_id INTEGER
)
```

Golden Set 用于 Regression 测试：定期运行同一组输入，对比输出是否符合预期。

<!-- @end-section -->

<!-- @section: mcp-audit -->

## 5. MCP 调用审计

### 5.1 MCP 调用日志

每次 MCP 工具调用记录到 `mcp_call_log` 表：

```sql
CREATE TABLE mcp_call_log (
  id          INTEGER PRIMARY KEY,
  agent_name  TEXT,
  tool_name   TEXT,
  input       TEXT,   -- JSON
  output      TEXT,   -- JSON
  success     INTEGER,
  error       TEXT,
  duration_ms INTEGER,
  workspace_id INTEGER,
  tenant_id   INTEGER,

  -- Ed25519 签名字段（Migration 050）
  payload_hash TEXT,  -- SHA-256(input + output + timestamp)
  signature    TEXT,  -- Ed25519 签名
  public_key   TEXT,  -- 签名用公钥

  created_at  INTEGER DEFAULT (unixepoch())
)
```

### 5.2 不可篡改签名（Ed25519）

Migration 050 为 MCP 调用日志添加 Ed25519 签名：

```
payload_hash = SHA-256(tool_name + input + output + timestamp)
signature    = Ed25519.sign(payload_hash, privateKey)
public_key   = Ed25519.getPublicKey(privateKey)
```

**用途**：提供可验证的调用记录，防止日志被篡改，支持合规审计。

### 5.3 Agent 运行记录（Runs 表）

Migration 046 引入 `runs` 表，提供完整的 Agent 运行 provenance：

```sql
CREATE TABLE runs (
  id          INTEGER PRIMARY KEY,
  agent_id    INTEGER,
  run_hash    TEXT UNIQUE,   -- SHA-256(inputs + timestamp)
  lineage     TEXT,          -- JSON：来自哪个 run，派生关系
  signed_by   TEXT,          -- 签名方（代理名或系统）
  signature   TEXT,          -- 运行摘要签名
  status      TEXT,
  inputs      TEXT,          -- JSON
  outputs     TEXT,          -- JSON
  started_at  INTEGER,
  completed_at INTEGER,
  workspace_id INTEGER
)
```

**lineage 追踪**：记录 run 的来源链（如 run B 由 run A 派生），支持因果分析。

<!-- @end-section -->

<!-- @section: injection-detection -->

## 6. 注入检测

### 6.1 消息发送时的安全扫描

`POST /api/agents/message` 在实际投递前执行安全扫描：

**注入检测**（blocklist 关键词/模式）：
- "ignore previous instructions"
- "disregard all prior"
- "system prompt:"
- `<|system|>` token 注入
- Base64 编码的恶意指令

**分级响应**：
- Critical 级别注入 → 阻断，返回 400，写 security_events（injection.attempt）
- Warning 级别 → 允许通过，写 security_events（warning），前端提示

**秘密泄露扫描**（防止意外泄露 API Key）：
- 检测消息内容中的 API Key 格式字符串
- 发现后写 security_events（secret.exposure）
- 不阻断但触发信任分扣减（-0.20 最重的惩罚）

### 6.2 技能安全扫描

见 [[04-memory-skills-hub|§3.4 安全扫描]]，13 条规则覆盖提示注入、路径穿越、SSRF、凭证收集等攻击向量。

<!-- @end-section -->

<!-- @section: related -->

## 相关文档

- [[index|分析索引]]
- [[04-memory-skills-hub|04 记忆系统与技能 Hub]]
- [[06-data-models-api|06 数据模型与 API 参考]]
- [[07-insights|07 设计洞察与 Legion 参考]]

<!-- @end-section -->
