---
id: "analysis-mission-control-memory-skills-004"
title: "Mission Control 记忆系统与技能 Hub"
aliases: ["MC memory", "MC skills", "MC记忆技能"]
type: "analysis"
category: "design/analysis/mission-control"
tags: ["mission-control", "memory", "skills", "fts5", "knowledge-graph", "soul", "security-scan"]
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
---

# Mission Control 记忆系统与技能 Hub

<!-- @section: memory-system -->

## 1. 记忆系统

### 1.1 双轨存储架构

记忆系统采用**双轨并行存储**：

```
┌──────────────────────────────────────────────────────────┐
│              Memory Layer                                  │
│                                                           │
│  ┌─────────────────────┐    ┌──────────────────────────┐ │
│  │  文件系统（人类可读）  │    │  SQLite（向量/全文检索）   │ │
│  │                     │    │                          │ │
│  │  MEMORY_PATH/       │    │  memory_fts（FTS5虚拟表） │ │
│  │  ├── notes/         │    │  FTS5 Porter Stemmer     │ │
│  │  ├── projects/      │    │  BM25 排名               │ │
│  │  └── ...            │    │                          │ │
│  │  Markdown + YAML    │    │  {agentName}.sqlite      │ │
│  │  frontmatter        │    │  chunks 表（向量分块）    │ │
│  └─────────────────────┘    └──────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

**文件系统**：Markdown/txt 文件，存储于 `MEMORY_PATH`（`config.memoryDir`）。支持白名单路径前缀 `MEMORY_ALLOWED_PREFIXES`，防止目录穿越。路径安全由 `resolveSafeMemoryPath` + `isPathAllowed` 双重保护，符号链接一律跳过。

**Agent 向量记忆**：每个 Agent 独立的 SQLite 文件 `{openclawStateDir}/memory/{agentName}.sqlite`，包含 `chunks` 表（按文件+分块存储文本嵌入）。记忆图谱 API 读取各 Agent 的 `.sqlite` 文件，统计文件数/分块数/DB大小。

### 1.2 Schema 类型系统

文件以 YAML frontmatter 进行类型化，通过 `_schema` 块声明：

```yaml
---
_schema:
  type: note
  required: [title, tags]
  optional: [source]
title: "示例记忆条目"
tags: [research, architecture]
---

内容正文...
```

**WikiLink 提取**：自动识别 `[[链接目标]]` 语法，构建文件间连接图谱。

**Schema 验证**：读取时验证必填字段，验证失败产生 warning 但不阻塞操作。

### 1.3 FTS5 全文检索

**引擎**：SQLite FTS5 虚拟表，`porter unicode61` tokenizer（英文词干 + Unicode）

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS memory_fts USING fts5(
  path, title, content,
  tokenize='porter unicode61'
)
```

**BM25 排名**：
- 标题权重：**5.0**
- 路径/内容权重：1.0

**Snippet 提取**：`<mark>` 高亮标签，上下文 40 词

**查询语法支持**：
| 语法 | 示例 | 说明 |
|------|------|------|
| 简单词 | `legion` | 自动转前缀匹配 `legion*` |
| 多词 AND | `legion agent` | 隐式 AND |
| 精确短语 | `"model routing"` | 引号包围 |
| NEAR | `NEAR(agent task, 5)` | 5 词内共现 |
| OR / NOT | `agent OR skill NOT test` | 布尔运算 |
| 前缀 | `orche*` | 前缀通配符 |

**索引策略**：
- 懒惰初始化：首次搜索触发全量建索引
- 文件保存时增量更新单文件（`INSERT OR REPLACE`）
- 支持手动全量重建（`POST /api/memory/search?action=rebuild`）

### 1.4 权限控制

| 操作 | 最低角色 |
|------|---------|
| GET（树/内容/搜索） | viewer |
| POST（保存/创建） | operator |
| DELETE | admin |

**遍历限制**：深度 ≤ 8 层，文件 ≤ 1MB，最多扫描 2000 个文件。

### 1.5 知识处理管道（6 Rs 模型）

受 Ars Contexta "6 Rs" 处理模型启发，提供五个知识维护操作（`POST /api/memory/process`）：

| 操作 | 功能 | 算法细节 |
|------|------|---------|
| `reflect` | 发现孤立文件，生成链接建议 | 查找同目录下无互联文件，最多 20 条建议 |
| `reweave` | 检测陈旧文件 | 超过 14 天未更新但关联文件已更新 |
| `generate-moc` | 自动生成 Map of Content | 按目录分组，按连通度降序 |
| `gap-detect` | 检测知识缺口 | 断链(0.8) + 孤立(0.5) + 陈旧(0~1.0) + 被引但缺失(0.3+) |
| `consolidate` | 分析图拓扑 | 识别 Hub/Bridge/Cluster/Weak Edge |

**gap-detect 评分细节**：
- 断链（WikiLink 目标不存在）：0.8 分
- 孤立文件（无任何入边或出边）：0.5 分
- 陈旧文件（基于年龄动态 0~1.0）：`min(1.0, daysSinceUpdate / 365)`
- 被引多次但无对应文件：0.3 + 0.15 × 引用次数

**consolidate 图分析**：
- Hub 节点：度数 ≥ 均值 × 2
- Bridge 节点：进出邻域交叉 < 20%
- Cluster：BFS 连通分量 ≥ 3 节点
- Weak Edge：无共同邻居

### 1.6 记忆健康诊断

8 个维度，每项 0-100 分，均值为总分：

| 维度 | 计算方式 |
|------|---------|
| Schema 合规性 | frontmatter 有效文件比例 |
| Connectivity | 1 - 孤立文件比例 |
| Link Integrity | 1 - 断链率 |
| Freshness | 1 - 30 天陈旧率 |
| Atomicity | 1 - >10KB 文件比例（过大意味着混合内容）|
| Naming Conventions | snake_case/kebab-case 命名比例 |
| Organization | 1 - 根目录文件比例（应分类存放）|
| Description Quality | frontmatter 含 description 字段的比例 |

<!-- @end-section -->

<!-- @section: working-memory -->

## 2. Agent 工作记忆（Working Memory）

### 2.1 概念

Working Memory 是 Agent 的**运行时草稿本**，区别于持久知识库（Memory），它是短期、Agent 私有的运行状态记录。

### 2.2 存储

`agents.working_memory`（TEXT 字段，Migration 047）— 直接存储于 agents 表，随 Agent 更新。

**最大长度**：`MC_WORKING_MEMORY_MAX_BYTES`（默认 64KB），超长自动截断并写 warn 日志。

### 2.3 MCP 工具操作

| 工具 | 功能 |
|------|------|
| `mc_read_memory` | 读取当前工作记忆 |
| `mc_write_memory` | 写入（覆盖）工作记忆 |
| `mc_clear_memory` | 清空工作记忆 |

**追加模式**：`mc_write_memory` 支持 `append: true` 参数，在现有内容末尾追加而非覆盖。

<!-- @end-section -->

<!-- @section: skills-system -->

## 3. 技能 Hub（Skills Hub）

### 3.1 技能定义

技能（Skill）是存储于磁盘的 **Agent 能力模块**，以目录形式组织，目录内必须包含 `SKILL.md` 文件才被识别为有效技能。

`SKILL.md` 描述技能的用途、参数、使用示例等，Agent 可在运行时按需加载。

### 3.2 技能来源（7 类）

| source 标识 | 路径 | 优先级 |
|------------|------|-------|
| `user-agents` | `~/.agents/skills/` | 用户全局 |
| `user-codex` | `~/.codex/skills/` | 用户全局 |
| `project-agents` | `{cwd}/.agents/skills/` | 项目级 |
| `project-codex` | `{cwd}/.codex/skills/` | 项目级 |
| `openclaw` | `{OPENCLAW_STATE_DIR}/skills/` | 网关级 |
| `workspace` | `{OPENCLAW_STATE_DIR}/workspace/skills/` | 工作区共享 |
| `workspace-{agent}` | `workspace-{agentName}/skills/`（动态扫描）| Agent 私有 |

**去重策略**：同名技能按 source 优先级保留第一个发现的版本。

### 3.3 磁盘↔DB 双向同步

**同步策略**（disk wins）：

```
磁盘有、DB 无 → INSERT（新增）
磁盘有、DB 有、内容变化（SHA-256 hash 不同）→ UPDATE
DB 有、磁盘消失 → DELETE（仅限无 registry_slug 的本地技能）
```

**保留规则**：通过注册表（ClawdHub/skills.sh）安装的技能（`registry_slug` 非空）即使磁盘消失也不会从 DB 删除。

**触发频率**：调度器 `skill_sync` 任务每 60s 执行一次。

### 3.4 安全扫描

安装技能前执行 `checkSkillSecurity(content)`，基于 13 条规则：

**Critical（阻断安装）**：

| 规则 ID | 检测内容 |
|---------|---------|
| `prompt-injection-system` | "ignore previous instructions" 类提示注入 |
| `prompt-injection-role` | 角色操控/安全绕过指令 |
| `shell-exec-dangerous` | `rm -rf` / `curl|bash` / `eval()` |
| `data-exfiltration` | 数据外泄指令 |
| `path-traversal` | `../` 路径穿越 |
| `ssrf-internal-network` | 访问 localhost/内网地址 |
| `ssrf-metadata-endpoint` | AWS/GCP/Azure 元数据端点（169.254.169.254）|

**Warning（允许安装，标记）**：

| 规则 ID | 检测内容 |
|---------|---------|
| `credential-harvesting` | 硬编码 API key/secret/password |
| `obfuscated-content` | base64/十六进制编码内容 |
| `hidden-instructions` | HTML 注释中的可疑指令 |
| `excessive-permissions` | sudo/chmod 777 等 |

**Info（提示）**：

| 规则 ID | 检测内容 |
|---------|---------|
| `network-fetch` | 包含外部 HTTP URL 引用 |

### 3.5 外部注册表

**ClawdHub**（`clawhub.ai/api`）：
- SHA-256 内容完整性验证
- 安装前校验哈希值，防止内容篡改

**skills.sh**（`skills.sh/api`）：
- 标准 REST API

**Awesome OpenClaw**（GitHub README 解析）：
- 15 分钟内存缓存
- 正则解析 `- [name](url) - desc` 格式列表

**完整安装流程**：
```
fetch（从注册表拉取）
  │
  ▼ SHA-256 验证（ClawdHub）
  │
  ▼ checkSkillSecurity()
  │ Critical 风险 → 拒绝安装
  │ Warning 风险 → 安装但标记 security_status = 'warning'
  │
  ▼ 写磁盘（workspace/skills/{name}/SKILL.md）
  │
  ▼ 更新 DB（skills 表，registry_slug 标记来源）
```

<!-- @end-section -->

<!-- @section: exec-approvals -->

## 4. 执行审批系统

### 4.1 架构

执行审批是**人机协作的安全门控**，处理 Agent 工具执行的审批请求：

```
Agent 发起工具调用
  │
  ▼ Gateway 拦截 → 转发到 MC
POST /api/exec-approvals（来自 Gateway）
  │
  ▼ MC 前端显示审批请求
操作员审批
  │
  ▼ POST /api/exec-approvals/respond
     └─ Gateway 接收响应 → 放行或拒绝 Agent 的工具调用
```

### 4.2 三种审批决策

| 决策 | 效果 |
|------|------|
| `approve` | 批准本次执行 |
| `deny` | 拒绝本次执行 |
| `always_allow` | 永久允许该模式（写入 allowlist）|

### 4.3 Allowlist 机制

**持久化路径**：`{openclawHome}/exec-approvals.json`（权限 0o600）

```json
{
  "version": 1,
  "agents": {
    "{agentId}": {
      "allowlist": [
        { "pattern": "bash:*" },
        { "pattern": "read_file:/workspace/**" }
      ]
    }
  }
}
```

**并发控制**：SHA-256 CAS（Compare-And-Swap）乐观并发，并发写入冲突返回 409 CONFLICT。

**模式匹配**：`{工具类型}:{参数模式}` 格式，支持通配符 `*`。

<!-- @end-section -->

<!-- @section: related -->

## 相关文档

- [[index|分析索引]]
- [[02-task-agent-lifecycle|02 任务与 Agent 生命周期]]
- [[05-security-eval-framework|05 安全框架与 Agent 评估]]
- [[07-insights|07 设计洞察与 Legion 参考]]

<!-- @end-section -->
