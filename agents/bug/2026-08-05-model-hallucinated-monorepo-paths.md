---
id: "bug-explore-hallucinated-paths-001"
title: "BUG — 探索项目时模型臆想 monorepo 路径，反复 list_files/read_file 不存在文件"
aliases: ["hallucinated monorepo paths", "list_files 不存在路径", "探索臆想路径"]
type: "reference"
category: "agents/bug"
tags: ["bug", "agent-behavior", "tool-loop", "hallucination", "list_files", "read_file", "truncation", "grounding"]
version: "1.0.0"
created: "2026-08-05"
updated: "2026-08-05"
author: "jxncyjq"
status: "draft"
parent: null
children: []
related_docs:
  - id: "spec-agent-runtime-001"
    relation: "references"
    path: "../../design/architecture/agent_components/agent-runtime-spec.md"
  - id: "deepthinking-interop-009"
    relation: "related_to"
    path: "../../design/deepthinking/09-agent-interop-protocols-status.md"
---

# BUG — 探索项目时模型臆想 monorepo 路径，反复 list_files/read_file 不存在文件

<!-- @section: summary -->
## 摘要

- **会话**：`session-1785858573220921200`（2026-08-04 15:49–16:24 UTC，default-agent）
- **关键任务**：`gui-task-1785860455782742300`，输入 = *"请熟悉和探索当前项目的架构，并详细把其中关于画布相关的流程列出"*
- **工作区**：`F:\source\a2m\smart-print-design`（**单包 Vue/Vite 应用**，代码在 `src/`，**无 `packages/`，无 `pnpm-workspace.yaml`，非 monorepo**）
- **现象**：模型反复对**不存在的路径** `packages/canvas-lark`、`packages/a2c`、`packages/a2m-editor`、`apps/xprint-design` 调 `list_files`/`read_file`，全部失败；探索任务无法完成，模型坦白 *"工具连续失败/被截断……候选路径"*。
- **判定**：**非工具 bug**。工具行为正确（如实报告路径不存在）；根因是**模型未 grounding 就臆想布局**，叠加上下文折叠与含糊错误文案，形成"猜-败-猜"循环。
<!-- @end-section -->

<!-- @section: evidence -->
## 证据

1. **用户没提任何包名**：任务输入只要求"探索架构 + 列画布流程"。`packages/canvas-lark` 等名字**全出现在模型自己的输出**里，标为"候选相关包/应用"。
2. **工作区没有这些名字**：`grep -rniE "a2c|a2m-editor|canvas-lark"` 全工作区，只命中 `pnpm-lock.yaml` 的 integrity 哈希与 `colors.json` 的偶然子串，**无真实包名**。
3. **名字是从词汇猜的**：项目名 `a2m` → `a2c`/`a2m-editor`；任务词"画布/canvas" → `canvas-lark`；`smart-print-design` → `xprint-design`。假设成 `packages/*` + `apps/*` monorepo。
4. **工具失败原文**（`debug inference message` role=tool）：
   ```
   no files
   skipped .: GetFileAttributesEx F:\source\a2m\smart-print-design\packages\canvas-lark: The system cannot find the path specified.
   ...\packages\a2c: The system cannot find the path specified.
   ...\packages\a2m-editor: The system cannot find the path specified.
   ```
5. **模型从没成功列出真实结构**：该会话所有 tool 结果里**无** `src/App.vue`/`package.json`/`vite.config` 等真实文件。
6. **截断**：同任务触发 **50 次 `[older tool output trimmed: N chars]`**（合计约 48k 字被折），因累积 tool 输出超 `defaultMaxPromptChars = 16000`（`runtime/runtime.go:161`），`conversation.render`（`runtime/messages.go:125`）从最老 tool 结果起折叠成桩。
<!-- @end-section -->

<!-- @section: rootcause -->
## 根因

**模型未先 grounding（`list_files(".")` 确认真实根）就按名字臆想 monorepo 路径**，并被两个因素放大：

1. **上下文折叠抹掉真实锚点**：早期若拿到过真实结果（idx3≈4360 字、idx5/7≈3242 字，均被折成桩），16000 预算下很快被 `render` 折叠 → 模型丢失真实结构 → 继续猜 → 更多失败 → 更多输出 → 更多折叠（恶性循环）。
2. **工具错误文案含糊、不引导**：`list_files` 对不存在目录只回 `no files` + Windows `GetFileAttributesEx ... cannot find the path specified`，**没告诉模型"该路径不存在，请先列根目录"**，模型不知错在哪，反复换名字猜。
3. **monorepo 先入为主**：`agents.md` 泛泛提到"workspaces、monorepo（Turborepo/Nx）"作为通用能力描述，可能强化了错误假设（非该项目实况）。
<!-- @end-section -->

<!-- @section: impact -->
## 影响

- 探索任务失败，模型无法可靠产出，坦白有编造风险（已诚实止损，未硬编）。
- 大量无效 tool 轮次堆积 → 触发 50 次上下文折叠 / 48k 字被折 → token 浪费 + 真实上下文被挤掉。
<!-- @end-section -->

<!-- @section: fix -->
## 修复方向

1. **【进行中】`list_files`/`read_file` 对不存在路径返回明确引导**：把含糊的 `no files` + OS 错，改成 *"目录 %q 不存在于工作区根；请先 `list_files`（不带 directory 或 directory="."）查看真实布局"*（`list_files` 见 `internal/tool/builtin.go:943 listFilesTool`，`guard.Check` 不验存在 → `WalkDir` 首 call 报 OS 错）。分支 `fix/list-files-missing-dir-guidance`（worktree，基 origin/master）。
   - 直接打断"猜-败-猜"循环，逼模型 grounding。
2. **（可选）折叠策略保护结构类结果**：`render` 折叠时优先保留最近一次成功的目录/结构列表，避免真实布局被过早折成桩；或按需抬 `MaxPromptChars`。
3. **（可选）prompt/persona 层引导**：探索任务先 `list_files(".")` grounding，禁止在未确认真实布局前对猜测路径批量读。
<!-- @end-section -->

## 相关文档

- [[spec-agent-runtime-001|AgentRuntime 组件规范]] — tool-loop / prompt 预算 / render 折叠
- [[deepthinking-interop-009|Agent 互操作协议实现现状]] — 同期分析（不同主题，同调查批次）
