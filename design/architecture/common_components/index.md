---
id: "index-common-components"
title: "公共组件规范索引"
aliases: ["公共组件索引", "common-components-index"]
type: "index"
category: "design/architecture/common_components"
tags: ["index", "component-spec", "common", "agent-engine", "maas", "llm-wiki", "legion"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
related_docs:
  - id: "spec-agent-component-registry-000"
    relation: "sibling"
    path: "../agent_components/agent-component-registry.md"
  - id: "index-components"
    relation: "sibling"
    path: "../maas_components/index.md"
  - id: "index-know-components"
    relation: "sibling"
    path: "../know_components/index.md"
  - id: "spec-platform-component-registry-000"
    relation: "referenced_by"
    path: "../platform-component-registry.md"
---

<!-- @section: overview -->
# 公共组件规范索引

本目录收录 Legion **MaaS 层、Agent 引擎层与 LLM Wiki / Knowledge Graph 层共同依赖的基础设施组件**规范文档。

这些组件不归属于 MaaS、Agent 或 Know 任意一侧，而是跨模块共用的平台能力：

| 特征 | 说明 |
|------|------|
| 跨层共享 | MaaS、Agent Engine、Knowledge Graph 均可依赖，通过 `Arc` 注入 |
| 无业务逻辑 | 仅提供基础平台能力（事件、向量、审计） |
| 独立注册 | 独立于两侧的 ComponentRegistry，构造时注入 |
| Noop 降级 | 均提供 Noop 实现，支持测试和 minimal 装配 |

依赖关系总览请查阅：
→ **[[../platform-component-registry|平台组件注册表与跨模块依赖]]**

<!-- @end-section -->

---

<!-- @section: component-table -->
## 一、公共组件一览

| ID  | 组件 | 规范文档 | 说明 |
|-----|------|----------|------|
| X00 | EventBus | [event-bus-spec.md](./event-bus-spec.md) | 异步发布/订阅事件总线：流式 token 推送、任务状态广播、学习事件传递 |
| X01 | EmbeddingProvider | [embedding-provider-spec.md](./embedding-provider-spec.md) | 向量嵌入服务：情景记忆检索、知识语义搜索、语义缓存 |
| X02 | ImmutableAuditLog | [immutable-audit-log-spec.md](./immutable-audit-log-spec.md) | 不可变审计日志：仅 INSERT，SHA-256 内容哈希，支持 HardLoop 和知识治理证据链 |
| X03 | SafeFetcher | [safe-fetcher-spec.md](./safe-fetcher-spec.md) | 安全 URL 抓取：scheme 限制、SSRF 防护、redirect 复检、大小上限 |
| X04 | PathGuard | [path-guard-spec.md](./path-guard-spec.md) | 路径守卫：规范化、路径穿越防护、敏感文件识别、输出目录沙盒 |
| X05 | OutputSanitizer | [output-sanitizer-spec.md](./output-sanitizer-spec.md) | 输出净化：HTML/YAML/Markdown/MCP 文本转义与控制字符清理 |

<!-- @end-section -->

---

<!-- @section: usage -->
## 二、各组件使用方

### X00 EventBus

**MaaS 侧使用**：
- `TelemetryEmitter`（C19）发布计费/指标事件
- `StreamProxy`（C50）发布流式 token 事件
- `MaasInferenceClient`（C70）可把推理端口事件统一转发

**Agent Engine 侧使用**：
- `AgentRuntime`（A01）：每个 token delta 推送（流式 LLM 调用）、发布 `LearningEvent`
- `AgentCoordinator`（A02）：任务状态变更广播（`task.in_progress`、`task.suspended` 等）
- `TrustScoreManager`（A61）：信任分变更事件
- `AegisReviewer`（A60）：审核结果广播

**Knowledge Graph 侧使用**：
- `KnowledgeGovernance`（K50）：审核、冲突、裁决事件广播
- `WikiReportGenerator`（K60）：报告刷新通知

### X01 EmbeddingProvider

**使用者**：
- `MemoryProvider`（A40）：P5 相关经验记忆 TopK=5 检索
- `EpisodicMemoryStore`（A42）：情景记忆向量索引构建与检索
- `VectorStore`（K32）：知识块向量检索
- `SemanticCache`（C16）：语义缓存

**缺失时降级**：EmbeddingProvider 缺失时，向量检索返回空结果，`MemoryProvider` 退化为 FTS5 全文检索。

### X02 ImmutableAuditLog

**使用者**：
- `AgentCoordinator`（A02）：每次心跳写入审计条目
- `TrustScoreManager`（A61）：安全事件记录
- `EvolutionEventLog`（A54）：进化操作审计
- HardLoop 响应链（§6.2.1）：触发时间、ratio 值、工具调用证据写入
- `KnowledgeStore`（K30）/ `KnowledgeGovernance`（K50）：知识状态转换、冲突裁决、审核证据链
- `AuditLogger`（C62）：可作为 MaaS 推理审计适配器，将摘要镜像到不可变证据链

### X03 SafeFetcher

**使用者**：
- `ContentIngestor`（K11）：URL、arXiv、网页、外部资料抓取

**缺失时降级**：禁止外部 URL 接入，只允许本地文件进入知识库。

### X04 PathGuard

**使用者**：
- `CorpusDetector`（K10）：扫描根路径和 symlink 安全校验
- `ContentIngestor`（K11）：converted sidecar 输出路径限定
- `CodeStructureExtractor`（K12）：源文件路径规范化
- `KnowledgeGraphMCP`（K61）：只允许读取知识图谱产物目录

**缺失时降级**：生产环境不允许缺失；测试环境可使用临时目录限定实现。

### X05 OutputSanitizer

**使用者**：
- `SemanticExtractor`（K13）：LLM 输出标签清理
- `KnowledgeGraphBuilder`（K20）：node label / edge context 规范化
- `WikiReportGenerator`（K60）：Markdown/HTML/YAML 报告转义
- `KnowledgeGraphMCP`（K61）：MCP 文本输出净化

<!-- @end-section -->

---

<!-- @section: noop-behaviors -->
## 三、Noop 降级行为

| 组件 | 缺失时 Noop 行为 | 影响 |
|------|----------------|------|
| EventBus（X00） | 所有 `publish` 调用为空操作；`subscribe` 返回不推送的哑频道 | UI 不收到流式 token；后台学习事件不传递（GEP 不触发）|
| EmbeddingProvider（X01） | `embed` 返回零向量；向量检索返回空结果 | 记忆检索退化为 FTS5；P5 经验记忆注入为空 |
| ImmutableAuditLog（X02） | 审计条目写入 stdout（仅开发环境） | 无持久化审计链；HardLoop 证据不落盘 |
| SafeFetcher（X03） | 拒绝所有外部 URL | 外部网页/arXiv 接入不可用 |
| PathGuard（X04） | 仅允许测试临时目录 | 生产必须启用，否则拒绝启动 |
| OutputSanitizer（X05） | 只做长度截断 | HTML/YAML/MCP 输出风险升高，生产不建议 |

> ⚠️ ImmutableAuditLog 的 Noop 仅用于开发/测试。生产环境 **必须** 使用持久化实现，否则无法满足合规要求。

<!-- @end-section -->

---

<!-- @section: quick-reference -->
## 四、快速入口

| 需求 | 文档 |
|------|------|
| 了解 Agent 全量依赖关系 | [../agent_components/agent-component-registry.md](../agent_components/agent-component-registry.md) |
| 了解平台跨模块依赖 | [../platform-component-registry.md](../platform-component-registry.md) |
| 配置事件订阅与流式推送 | [event-bus-spec.md](./event-bus-spec.md) |
| 配置记忆向量检索 | [embedding-provider-spec.md](./embedding-provider-spec.md) |
| 配置不可变审计链 | [immutable-audit-log-spec.md](./immutable-audit-log-spec.md) |
| 配置安全外部抓取 | [safe-fetcher-spec.md](./safe-fetcher-spec.md) |
| 配置路径沙盒 | [path-guard-spec.md](./path-guard-spec.md) |
| 配置输出净化 | [output-sanitizer-spec.md](./output-sanitizer-spec.md) |
| 了解 MaaS 侧组件 | [../maas_components/index.md](../maas_components/index.md) |
| 了解 Knowledge Graph 侧组件 | [../know_components/index.md](../know_components/index.md) |

<!-- @end-section -->
