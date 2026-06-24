---
id: "analysis-mission-control-data-api-006"
title: "Mission Control 数据模型与 API 参考"
aliases: ["MC schema", "MC API", "MC数据模型"]
type: "analysis"
category: "design/analysis/mission-control"
tags: ["mission-control", "schema", "rest-api", "mcp-tools", "database", "openapi"]
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
  - id: "analysis-mission-control-task-agent-002"
    relation: "related_to"
    path: "./02-task-agent-lifecycle.md"
---

# Mission Control 数据模型与 API 参考

<!-- @section: database-schema -->

## 1. 完整数据库 Schema

数据库使用 SQLite（better-sqlite3），WAL 模式，时间戳均为 Unix 整型秒。45 个 migration 文件，实际编号有跳过（030/031），共约 50 张表。

### 1.1 核心业务表

#### tasks（任务看板核心）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PK | 自增主键 |
| title | TEXT | 任务标题 |
| description | TEXT | 详细描述 |
| status | TEXT | inbox/assigned/in_progress/quality_review/done |
| priority | TEXT | low/medium/high/urgent/critical |
| assigned_to | TEXT | Agent 名称（字符串，非 FK）|
| created_by | TEXT | 创建者 |
| project_id | INTEGER FK | 所属项目 |
| project_ticket_no | INTEGER | 项目内递增序号 |
| outcome | TEXT | 任务结果摘要（M026）|
| error_message | TEXT | 失败原因（M026）|
| resolution | TEXT | 解决方案（M026）|
| feedback_rating | INTEGER | 1-5 分评价（M026）|
| feedback_notes | TEXT | 评价备注（M026）|
| retry_count | INTEGER | 手动重试次数（M026）|
| completed_at | INTEGER | 首次完成时间（M026，一旦设定不变）|
| github_issue_number | INTEGER | 关联 GitHub Issue（M028）|
| github_repo | TEXT | GitHub 仓库（M028）|
| github_synced_at | INTEGER | 最后同步时间（M028）|
| github_branch | TEXT | 关联分支（M028）|
| github_pr_number | INTEGER | 关联 PR 号（M028）|
| github_pr_state | TEXT | PR 状态（M028）|
| dispatch_attempts | INTEGER | 调度尝试次数（M045）|
| tags | TEXT | JSON 数组 |
| metadata | TEXT | JSON（含 recurrence 配置）|
| workspace_id | INTEGER FK | 工作区隔离 |
| created_at | INTEGER | 创建时间 |
| updated_at | INTEGER | 更新时间 |

#### agents（Agent 管理）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PK | 自增主键 |
| name | TEXT UNIQUE | Agent 名称（唯一标识）|
| role | TEXT | 角色（developer/reviewer/researcher 等）|
| session_key | TEXT UNIQUE | 会话密钥（用于 Gateway 通信）|
| soul_content | TEXT | SOUL.md 内容（人设配置）|
| status | TEXT | offline/idle/busy/sleeping/error |
| last_seen | INTEGER | 最后心跳时间 |
| last_activity | INTEGER | 最后活动时间 |
| config | TEXT | JSON（capabilities/framework 等）|
| source | TEXT | manual/local/sync 等（M034）|
| content_hash | TEXT | SOUL 内容哈希（M034）|
| workspace_path | TEXT | Agent 工作区路径（M034）|
| hidden | INTEGER | 软隐藏标志（M042）|
| working_memory | TEXT | 运行时草稿本（M047，最大 64KB）|
| runtime_type | TEXT | 如 hermes（M049）|
| workspace_id | INTEGER FK | 工作区隔离 |
| created_at | INTEGER | 注册时间 |

#### comments（任务讨论）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PK | — |
| task_id | INTEGER FK | 关联任务 |
| author | TEXT | 评论者（Agent/用户名）|
| content | TEXT | 评论内容 |
| parent_id | INTEGER | 自引用（嵌套回复）|
| mentions | TEXT | JSON 数组（@提及名称列表）|

#### activities（实时活动流）

| 字段 | 类型 | 说明 |
|------|------|------|
| type | TEXT | task_created/task_updated/agent_status_change 等 |
| entity_type | TEXT | task/agent/comment |
| entity_id | TEXT | 实体 ID |
| actor | TEXT | 操作者 |
| description | TEXT | 人类可读描述 |
| data | TEXT | JSON 附加数据 |

### 1.2 认证与用户表

#### users

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PK | — |
| username | TEXT UNIQUE | 用户名 |
| display_name | TEXT | 显示名 |
| password_hash | TEXT | scrypt 哈希（仅 local provider）|
| role | TEXT | viewer/operator/admin |
| provider | TEXT | local/google/proxy |
| provider_user_id | TEXT | OAuth 提供商用户 ID |
| email | TEXT | 邮箱（OAuth 用）|
| avatar_url | TEXT | 头像 URL |
| is_approved | INTEGER | 1=已批准（Google OAuth 需审批）|
| approved_by | TEXT | 审批人 |
| approved_at | INTEGER | 审批时间 |
| workspace_id | INTEGER FK | — |

#### user_sessions

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PK | — |
| token | TEXT UNIQUE | SHA-256(rawToken)（M043 安全升级）|
| user_id | INTEGER FK | — |
| expires_at | INTEGER | 7 天有效期 |
| ip_address | TEXT | 登录 IP |
| user_agent | TEXT | 浏览器信息 |
| workspace_id | INTEGER FK | — |
| tenant_id | INTEGER FK | — |

#### api_keys（用户级）

| 字段 | 类型 | 说明 |
|------|------|------|
| user_id | INTEGER FK | 所属用户 |
| key_hash | TEXT | SHA-256(rawKey) |
| key_prefix | TEXT | 展示用前缀（如 `mc-...`）|
| label | TEXT | 描述标签 |
| role | TEXT | 关联角色 |
| scopes | TEXT | JSON 数组权限范围 |
| expires_at | INTEGER | 可选过期时间 |
| is_revoked | INTEGER | 吊销标志 |

#### agent_api_keys（Agent 专属最小权限）

| 字段 | 类型 | 说明 |
|------|------|------|
| agent_id | INTEGER FK | 所属 Agent |
| key_hash | TEXT | SHA-256(rawKey) |
| key_prefix | TEXT | 展示用前缀 |
| scopes | TEXT | JSON 数组（admin/operator/viewer）|
| expires_at | INTEGER | 可选过期时间 |
| revoked_at | INTEGER | 吊销时间 |

### 1.3 工作流与编排表

#### workflow_templates

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PK | — |
| name | TEXT | 模板名称 |
| model | TEXT | 使用的 LLM 模型 |
| task_prompt | TEXT | 任务提示词模板 |
| timeout_seconds | INTEGER | 超时时间 |
| use_count | INTEGER | 使用次数 |

#### workflow_pipelines / pipeline_runs

```sql
-- workflow_pipelines
steps TEXT  -- JSON 数组：[{template_id, on_failure: 'stop'|'continue'}, ...]

-- pipeline_runs  
status TEXT            -- pending/running/completed/failed/cancelled
current_step INTEGER   -- 当前执行步骤索引
steps_snapshot TEXT    -- JSON（执行快照，含 spawn_id）
triggered_by TEXT      -- 触发来源
```

#### projects

| 字段 | 类型 | 说明 |
|------|------|------|
| name | TEXT | 项目名 |
| slug | TEXT | URL 友好名称 |
| ticket_prefix | TEXT | 工单号前缀（如 `MC`）|
| ticket_counter | INTEGER | 原子递增计数器 |
| github_repo | TEXT | 关联 GitHub 仓库 |
| github_sync_enabled | INTEGER | 是否启用同步 |
| github_default_branch | TEXT | 默认分支 |
| deadline | INTEGER | 截止日期 |
| color | TEXT | 项目颜色标识 |

### 1.4 安全与审计表

#### security_events

| 字段 | 类型 | 说明 |
|------|------|------|
| event_type | TEXT | 事件类型（如 injection.attempt）|
| severity | TEXT | info/warning/critical |
| source | TEXT | 来源系统/模块 |
| agent_name | TEXT | 相关 Agent |
| detail | TEXT | 详细描述 |
| ip_address | TEXT | 来源 IP |
| workspace_id | INTEGER FK | — |
| tenant_id | INTEGER FK | — |

#### agent_trust_scores

见 [[05-security-eval-framework|§1.3]] 详细字段说明。

#### mcp_call_log

| 字段 | 类型 | 说明 |
|------|------|------|
| agent_name | TEXT | 调用 Agent |
| tool_name | TEXT | 工具名称 |
| input | TEXT | JSON 输入 |
| output | TEXT | JSON 输出 |
| success | INTEGER | 是否成功 |
| error | TEXT | 错误信息 |
| duration_ms | INTEGER | 执行耗时 |
| payload_hash | TEXT | SHA-256 内容哈希（M050）|
| signature | TEXT | Ed25519 签名（M050）|
| public_key | TEXT | 签名公钥（M050）|

#### runs（完整 Agent 运行 Provenance）

| 字段 | 类型 | 说明 |
|------|------|------|
| run_hash | TEXT UNIQUE | SHA-256(inputs + timestamp) |
| lineage | TEXT | JSON 派生关系链 |
| signed_by | TEXT | 签名方 |
| signature | TEXT | 运行摘要签名 |
| status | TEXT | 运行状态 |
| inputs | TEXT | JSON 输入集 |
| outputs | TEXT | JSON 输出集 |

### 1.5 多租户表

#### tenants

| 字段 | 类型 | 说明 |
|------|------|------|
| name | TEXT | 租户名称 |
| slug | TEXT | URL 唯一标识 |
| owner_gateway | TEXT | 默认网关地址 |
| status | TEXT | 租户状态 |
| config | TEXT | JSON 配置 |

#### workspaces（工作区隔离）

```
Tenant (1) ─── (N) Workspace (1) ─── (N) Users/Agents/Tasks/...
```

Migration 021-023 为所有核心表添加 `workspace_id` 字段，实现数据隔离。

### 1.6 技能与记忆表

#### skills

| 字段 | 类型 | 说明 |
|------|------|------|
| name | TEXT | 技能名称 |
| source | TEXT | 来源路径标识 |
| path | TEXT | 磁盘路径 |
| registry_slug | TEXT | 注册表标识（非空表示从注册表安装）|
| security_status | TEXT | safe/warning/critical |
| content_hash | TEXT | SHA-256 内容哈希 |

#### memory_fts（FTS5 虚拟表）

```sql
CREATE VIRTUAL TABLE memory_fts USING fts5(
  path, title, content,
  tokenize='porter unicode61'
)
```

### 1.7 其他系统表

| 表名 | 用途 |
|------|------|
| notifications | 通知（recipient/type/read_at）|
| task_subscriptions | 任务订阅（agent_name + task_id）|
| messages | Agent 间消息（conversation_id/from_agent/to_agent）|
| webhooks | 出站 Webhook 配置 |
| webhook_deliveries | Webhook 投递历史（含重试状态）|
| settings | key-value 配置（分 category）|
| alert_rules | 自动告警规则 |
| audit_log | 操作审计日志 |
| gateways | 网关配置 |
| gateway_health_logs | 网关健康探测日志 |
| direct_connections | Agent 直连通道 |
| github_syncs | GitHub 同步历史 |
| token_usage | Token 计费记录（input/output/cost_usd）|
| claude_sessions | Claude Code 会话扫描记录 |
| adapter_configs | 框架适配器配置 |
| eval_runs | 评估运行记录 |
| eval_golden_sets | 评估黄金数据集 |
| eval_traces | 评估追踪记录 |
| spawn_history | Agent 进程启动历史 |
| access_requests | Google OAuth 待审批请求 |
| standup_reports | 每日站会报告 |
| provision_jobs | 租户配置作业 |
| provision_events | 租户配置事件 |
| project_agent_assignments | 项目-Agent 关联 |

<!-- @end-section -->

<!-- @section: rest-api -->

## 2. REST API 端点（101 个）

OpenAPI 规范：`openapi.json`（版本 1.3.0），14 个标签分组。

### 2.1 Agent 管理（agents 标签）

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | /api/agents | 列出所有 Agent（支持 workspace 过滤）| viewer |
| POST | /api/agents | 创建 Agent | operator |
| PUT | /api/agents | 更新 Agent 状态/配置 | operator |
| GET | /api/agents/[id] | 获取单个 Agent | viewer |
| DELETE | /api/agents/[id] | 删除 Agent | admin |
| POST | /api/agents/register | 幂等注册（5次/min 限流）| viewer |
| GET | /api/agents/[id]/heartbeat | 心跳（GET 轮询）| viewer |
| POST | /api/agents/[id]/heartbeat | 心跳（POST 增强版）| operator |
| GET | /api/agents/[id]/soul | 读取 SOUL | viewer |
| PUT | /api/agents/[id]/soul | 写入 SOUL | operator |
| PATCH | /api/agents/[id]/soul | 列出模板 | viewer |
| GET | /api/agents/comms | Agent 间通信图谱 | viewer |
| POST | /api/agents/message | 向 Agent 发送消息 | operator |
| GET | /api/agents/[id]/evals | Agent 评估报告 | viewer |
| GET | /api/agents/[id]/evals/optimize | 优化建议 | viewer |

### 2.2 任务管理（tasks 标签）

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | /api/tasks | 列出任务（status/assigned_to/project 过滤）| viewer |
| POST | /api/tasks | 创建任务 | operator |
| GET | /api/tasks/[id] | 获取任务详情 | viewer |
| PUT | /api/tasks/[id] | 更新任务（状态保护）| operator |
| DELETE | /api/tasks/[id] | 删除任务 | admin |
| GET | /api/tasks/queue | 原子拉取队列任务 | operator |
| GET | /api/tasks/[id]/comments | 获取评论列表 | viewer |
| POST | /api/tasks/[id]/comments | 添加评论 | operator |
| GET/PUT | /api/tasks/[id]/quality-review | 质量审查操作 | operator |

### 2.3 记忆与知识（memory 标签）

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | /api/memory | 列出/读取文件（tree/content）| viewer |
| POST | /api/memory | 保存文件 | operator |
| DELETE | /api/memory | 删除文件 | admin |
| GET | /api/memory/graph | Agent 记忆图谱 | viewer |
| GET | /api/memory/search | FTS5 全文搜索 | viewer |
| POST | /api/memory/search | 重建搜索索引 | admin |
| POST | /api/memory/process | 知识处理（reflect/reweave/moc/gap/consolidate）| operator |
| GET | /api/memory/health | 知识库健康诊断 | viewer |

### 2.4 技能管理（skills 标签）

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | /api/skills | 列出技能 | viewer |
| POST | /api/skills | 创建/上传技能 | operator |
| GET | /api/skills/[id] | 获取技能详情 | viewer |
| DELETE | /api/skills/[id] | 删除技能 | admin |
| GET | /api/skills/registry | 搜索外部注册表 | viewer |
| POST | /api/skills/registry | 从注册表安装 | operator |
| POST | /api/skills/sync | 手动触发磁盘同步 | admin |

### 2.5 安全（security 标签）

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | /api/security | 安全事件列表 | viewer |
| POST | /api/security | 写入安全事件 | operator |
| GET | /api/security/posture | 安全态势评分 | viewer |
| GET | /api/security/trust | Agent 信任分列表 | viewer |

### 2.6 评估（evals 标签）

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | /api/evals | 评估运行列表 | viewer |
| POST | /api/evals | 触发评估运行 | operator |
| GET | /api/evals/[runId] | 获取评估详情 | viewer |
| GET | /api/evals/golden-sets | 黄金数据集列表 | viewer |
| POST | /api/evals/golden-sets | 创建黄金数据集 | operator |

### 2.7 Token 与成本（tokens 标签）

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | /api/token-usage | Token 消耗统计 | viewer |
| POST | /api/token-usage | 上报 Token 使用 | operator |
| GET | /api/token-usage/by-agent | 按 Agent 汇总 | viewer |
| GET | /api/token-usage/timeline | 时间线数据 | viewer |

### 2.8 网关（gateways 标签）

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | /api/gateways | 列出网关 | viewer |
| POST | /api/gateways | 添加网关 | admin |
| GET/PUT/DELETE | /api/gateways/[id] | 单个网关 CRUD | admin |
| POST | /api/gateways/connect | 获取 WS 连接参数 | viewer |
| POST | /api/gateways/health | 服务端健康探测 | viewer |
| POST | /api/gateways/control | 控制网关进程 | admin |

### 2.9 其他端点分组

| 分组 | 主要端点 |
|------|---------|
| pipelines | CRUD + run（start/advance/cancel）|
| cron | list/logs/history + toggle/trigger/add/clone |
| webhooks | CRUD + test + reset_circuit |
| events | GET /api/events（SSE 流）|
| auth | login/logout/me/google/setup |
| admin/users | 用户管理 + 访问申请审批 |
| projects | CRUD + agent 分配 |
| exec-approvals | 获取待审批 + 响应（approve/deny/always_allow）|
| runs | Agent 运行记录 CRUD |

<!-- @end-section -->

<!-- @section: mcp-tools -->

## 3. MCP Server 工具（35 个）

**文件**：`scripts/mc-mcp-server.cjs`，零依赖（纯 Node.js 内置），stdio JSON-RPC 2.0

**接入方式**：
```bash
claude mcp add mission-control -- node /path/to/mission-control/scripts/mc-mcp-server.cjs
# 环境变量：MC_URL=http://127.0.0.1:3000  MC_API_KEY=<key>
```

### 3.1 Agent 管理类（6 个）

| 工具名 | 说明 |
|--------|------|
| `mc_list_agents` | 列出所有 Agent |
| `mc_get_agent` | 获取单个 Agent 详情 |
| `mc_heartbeat` | 发送心跳保活 |
| `mc_wake_agent` | 唤醒休眠 Agent |
| `mc_agent_diagnostics` | Agent 健康诊断 |
| `mc_agent_attribution` | 成本归因与审计轨迹（支持 24h/自定义窗口）|

### 3.2 记忆与知识类（10 个）

| 工具名 | 说明 |
|--------|------|
| `mc_read_memory` | 读取工作记忆 |
| `mc_write_memory` | 写入工作记忆（支持 append 模式）|
| `mc_clear_memory` | 清空工作记忆 |
| `mc_search_knowledge` | FTS5 全文搜索（支持 AND/OR/NOT/NEAR）|
| `mc_read_knowledge_file` | 读取知识文件（含 wiki-link 和 schema 验证）|
| `mc_write_knowledge_file` | 创建或更新知识文件 |
| `mc_knowledge_health` | 知识库健康度评分 |
| `mc_rebuild_search_index` | 重建全文搜索索引 |
| `mc_knowledge_gaps` | 检测知识缺口（断链/孤立/过时）|
| `mc_knowledge_consolidate` | 分析知识图谱拓扑（hub/bridge/cluster）|

### 3.3 SOUL 管理类（3 个）

| 工具名 | 说明 |
|--------|------|
| `mc_read_soul` | 读取 Agent SOUL |
| `mc_write_soul` | 写入 SOUL 或应用模板 |
| `mc_list_soul_templates` | 列出可用模板 |

### 3.4 任务管理类（8 个）

| 工具名 | 说明 |
|--------|------|
| `mc_list_tasks` | 列出任务（支持 status/agent/priority/search 过滤）|
| `mc_get_task` | 获取单任务 |
| `mc_create_task` | 创建任务 |
| `mc_update_task` | 更新任务（状态/优先级/指派等）|
| `mc_poll_task_queue` | 原子拉取任务（含 max_capacity 控制）|
| `mc_broadcast_task` | 向任务订阅者广播消息 |
| `mc_list_comments` | 获取任务评论 |
| `mc_add_comment` | 添加评论（支持线程回复）|

### 3.5 会话管理类（4 个）

| 工具名 | 说明 |
|--------|------|
| `mc_list_sessions` | 列出活跃会话 |
| `mc_control_session` | 控制会话（monitor/pause/terminate）|
| `mc_continue_session` | 向会话发送跟进 prompt |
| `mc_session_transcript` | 获取会话完整记录（含工具调用）|

### 3.6 Token/成本类（3 个）

| 工具名 | 说明 |
|--------|------|
| `mc_token_stats` | 聚合 Token 用量统计 |
| `mc_agent_costs` | 单 Agent 成本明细与时间线 |
| `mc_costs_by_agent` | N 天内按 Agent 汇总成本 |

### 3.7 其他工具（6 个）

| 工具名 | 说明 |
|--------|------|
| `mc_list_skills` | 技能库列表 |
| `mc_read_skill` | 读取技能详情 |
| `mc_list_cron` | 定时任务列表 |
| `mc_health` | 健康检查（无需鉴权）|
| `mc_dashboard` | 系统全局仪表板摘要 |
| `mc_status` | 系统状态（内存/磁盘/进程）|

### 3.8 Runs 类（3 个）

| 工具名 | 说明 |
|--------|------|
| `mc_list_runs` | Agent 运行记录列表 |
| `mc_get_run` | 获取运行详情 |
| `mc_create_run` | 创建运行记录 |

<!-- @end-section -->

<!-- @section: related -->

## 相关文档

- [[index|分析索引]]
- [[02-task-agent-lifecycle|02 任务与 Agent 生命周期]]
- [[03-orchestration-scheduler|03 编排引擎与调度器]]
- [[05-security-eval-framework|05 安全框架与 Agent 评估]]

<!-- @end-section -->
