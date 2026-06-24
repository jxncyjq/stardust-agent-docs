---
id: "analysis-graphify-security-testing-001"
title: "Graphify 安全与测试分析"
aliases: ["Graphify安全测试", "graphify security testing"]
type: "analysis"
category: "design/analysis/graphify"
tags: ["graphify", "security", "testing", "privacy"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
related_docs:
  - id: "analysis-graphify-index-001"
    relation: "parent"
    path: "./index.md"
---

# Graphify 安全与测试分析

## 1. 隐私边界

Graphify 的隐私模型是分层的：

| 输入 | 默认处理 | 是否出本机 |
|------|----------|------------|
| 代码 | tree-sitter 本地 AST | 否 |
| 音视频 | faster-whisper 本地转录 | 否 |
| 文档/PDF/图片 | LLM 语义提取 | 取决于后端 |
| URL | 下载后进入语料 | 会访问网络 |

README 明确说明：代码文件本地处理；文档、PDF、图片会发送给用户配置的 AI 后端；无 telemetry、无 usage tracking。

## 2. URL 与 SSRF 防护

`security.py` 对外部 URL 处理较认真：

- 只允许 `http` / `https`。
- 拒绝 `file://`、`ftp://`、`data:`。
- DNS 解析后拒绝 private、reserved、loopback、link-local、CGN 地址。
- 阻断云 metadata hostname。
- 自定义 redirect handler，重定向目标重新校验。
- fetch 时临时 patch `socket.getaddrinfo`，防 DNS rebinding。
- 下载按大小分块读取，有 50MB binary / 10MB text 默认上限。

这是一个值得借鉴的 SSRF 防线组合。

## 3. 路径与输出安全

`validate_graph_path()` 要求图文件必须位于 `graphify-out/` 内，防止 CLI/MCP 查询读取任意文件。

导出层也做了多处防护：

- `sanitize_label()` 去控制字符并限制长度。
- HTML 注入位置使用 escape。
- YAML front matter 字符串做转义，防止换行或控制字符注入新字段。
- HTML 可视化有节点数量上限，避免大图拖垮浏览器。

## 4. 敏感文件跳过

`detect.py` 默认跳过可能含 secret 的文件：

- `.env` / `.envrc`
- `.pem` / `.key` / `.p12` / `.pfx` / cert
- credential / secret / passwd / password / token / private_key
- ssh key、netrc、pgpass、htpasswd
- cloud credentials

这避免了最常见的“构图时把密钥发给 LLM”的事故。

## 5. 测试覆盖

测试目录按模块组织，覆盖面较广：

| 测试文件 | 关注点 |
|----------|--------|
| `test_pipeline.py` | 端到端 detect→extract→build→cluster→analyze→report→export |
| `test_extract.py` / `test_languages.py` | 多语言 AST、调用边、导入边、fixture |
| `test_detect.py` | 文件分类、ignore/include、manifest |
| `test_build.py` | schema 兼容、方向、去重、合并 |
| `test_cluster.py` | 社区检测与 cohesion |
| `test_analyze.py` / `test_report.py` | god nodes、surprises、报告结构 |
| `test_export.py` / `test_cli_export.py` | JSON/HTML/Obsidian/GraphML/SVG/Neo4j |
| `test_serve.py` / `test_query_cli.py` | MCP 与 CLI 查询 |
| `test_security.py` | URL、safe fetch、path guard、label sanitize |
| `test_watch.py` / `test_incremental.py` | watch、update、manifest 增量 |
| `test_llm_backends.py` / `test_ollama.py` | LLM 后端解析与配置 |

端到端测试刻意不走 LLM，使用 fixtures 做 AST-only pipeline，因此稳定、便宜、适合 CI。

## 6. 质量策略

Graphify 的测试策略有三个特点：

1. **模块单测 + pipeline 测试并存**：既测局部，又测模块之间的数据契约。
2. **fixtures 多语言覆盖**：每种语言至少有样例，降低 extractor 回归风险。
3. **网络和 LLM 大多 mock 或配置检测**：避免 CI 依赖外部服务。

## 7. 风险与改进建议

| 风险 | 建议 |
|------|------|
| `extract.py` 过大 | 按语言族拆分 extractor，保留统一 dispatcher |
| `__main__.py` 过大 | 拆成 `commands/install.py`、`commands/query.py`、`commands/extract.py` |
| LLM JSON 质量不稳定 | 引入更强 schema repair 和 per-chunk validation report |
| semantic cache 与 AST cache 分散 | 统一 cache manifest，记录 extractor version |
| inferred edges 可能误导 | 查询输出默认显示 confidence，并允许只看 `EXTRACTED` |

