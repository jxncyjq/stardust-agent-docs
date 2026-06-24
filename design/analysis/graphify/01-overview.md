---
id: "analysis-graphify-overview-001"
title: "Graphify 总览"
aliases: ["Graphify总览", "graphify overview"]
type: "analysis"
category: "design/analysis/graphify"
tags: ["graphify", "overview", "knowledge-graph"]
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

# Graphify 总览

## 1. 项目定位

Graphify 是一个 Python 包与 AI 助手技能集合，PyPI 包名为 `graphifyy`，CLI 命令为 `graphify`。它的目标不是替代搜索，而是为 AI 编程助手提供一个**项目级知识图谱记忆层**：

- 输入：代码、Markdown、PDF、图片、Office、Google Workspace、视频/音频、URL。
- 处理：本地 AST 提取 + LLM 语义提取 + 图构建 + 社区发现 + 报告生成。
- 输出：`graph.json`、`GRAPH_REPORT.md`、`graph.html`、Obsidian vault、Wiki、SVG、GraphML、Neo4j Cypher、MCP server。
- 消费者：Claude Code、Codex、OpenCode、Cursor、Copilot、Aider、Kiro、Gemini、Trae、OpenClaw 等 AI 编程环境。

## 2. 核心价值

Graphify 解决的是“AI 读大代码库成本高、容易迷路”的问题。它把一次性读取大量文件的成本前置为构图成本，后续查询通过图的局部遍历获取上下文。

关键能力：

| 能力 | 实现 |
|------|------|
| 代码结构理解 | `tree-sitter` 多语言 AST，提取类、函数、导入、调用、继承 |
| 文档语义理解 | LLM 后端抽取概念、引用、语义相似关系 |
| 图分析 | NetworkX + Leiden/Louvain 社区检测 |
| Agent 可消费 | `GRAPH_REPORT.md`、Wiki、MCP 工具、助手 hooks |
| 增量更新 | manifest + content hash + AST cache + semantic cache |
| 隐私分层 | 代码本地处理；文档/图片/PDF 可走 LLM |

## 3. 技术栈

| 层 | 主要依赖 |
|----|----------|
| 包管理 | setuptools，Python 3.10+ |
| 图结构 | NetworkX |
| AST | tree-sitter 与多语言 grammar 包 |
| 社区发现 | graspologic Leiden，可降级到 NetworkX Louvain |
| 去重 | datasketch MinHash/LSH + rapidfuzz Jaro-Winkler |
| LLM 后端 | Claude、Kimi、Gemini、OpenAI、Ollama、Bedrock |
| 可视化 | HTML + vis-network，SVG 可选 matplotlib |
| MCP | `mcp` optional dependency |
| 文档/媒体扩展 | pypdf、python-docx、openpyxl、faster-whisper、yt-dlp |

## 4. 关键结论

Graphify 的架构优点很明确：

1. **主流水线清晰**：每个阶段基本有独立模块，数据结构以普通 dict 与 NetworkX graph 传递，降低耦合。
2. **代码与语义分层**：代码走本地 deterministic AST，文档/图片/PDF 走 LLM，隐私与成本边界清楚。
3. **Agent 入口优秀**：不仅生成图，还把“使用图”的规则安装进各种助手环境。
4. **工程补丁密度高**：对 Windows、PowerShell、tree-sitter 版本、动态 import、tsconfig alias、增量合并、XSS/SSRF 等现实问题有大量修补。

主要风险：

1. `extract.py` 与 `__main__.py` 都偏大，呈现“功能聚合模块”趋势。
2. LLM semantic extraction 的质量依赖 prompt 与后端输出 JSON 稳定性。
3. 图边包含 `INFERRED` 与 `AMBIGUOUS`，适合导航和提出问题，但不能直接作为强事实源。
4. 增量 AST 更新会保留旧 semantic 节点/边，语义文件变化需要显式 LLM 更新。

