---
id: "bug-prompt-cache-backend-mismatch-001"
title: "BUG — prompt cache 断点机制与实际后端不匹配（cache_control 对 DeepSeek 无效）"
aliases: ["cache_control 无效", "StablePrefixLen 死代码", "prompt cache 后端错配", "DeepSeek 前缀缓存", "prompt_cache_hit_tokens 实测", "Kimi 缓存实测"]
type: "reference"
category: "agents/bug"
tags: ["bug", "prompt-cache", "deepseek", "anthropic", "cache-control", "stable-prefix", "capability-catalog", "cost", "measured", "kimi"]
version: "3.0.0"
created: "2026-08-16"
updated: "2026-08-17"
author: "jxncyjq"
status: "review"
parent: null
children: []
related_docs:
  - id: "design-legion-plugin-system-001"
    relation: "references"
    path: "../../design/architecture/legion-plugin-system.md"
  - id: "bug-explore-hallucinated-paths-001"
    relation: "related_to"
    path: "./2026-08-05-model-hallucinated-monorepo-paths.md"
---

# BUG — prompt cache 断点机制与实际后端不匹配

<!-- @section: overview -->
## 概述

Legion 实现了一套「稳定前缀 + 显式缓存断点」机制：`cognitive.BuildContext` 计算 `StablePrefixLen`，`adapter/http_maas.go` 据此在请求里发出 `cache_control: {"type": "ephemeral"}`。

**这是 Anthropic 的私有扩展。而 `agent.json` 里配置的后端全部是 DeepSeek 与 Kimi，两者都走服务端自动前缀缓存，都不需要该字段。**（两者的缓存语义**并不相同**——Kimi 给部分前缀匹配计分，DeepSeek 不给，见 §Kimi 实测。）

DeepSeek 的 Context Caching 是**服务端全自动前缀匹配**——没有 `cache_control` 参数、没有断点、没有 TTL 配置。**已用真实请求实测确认（见 §实测）：该字段被接受并忽略，HTTP 200，不报错也不生效。这条链路是死代码。**

影响不止于「白写了一段代码」：团队据此形成的缓存心智模型是断点语义，而真实语义是 token 级前缀匹配。**两种语义会导出方向相反的优化决策**——同一份数据在两种语义下的相反解读见 §影响。

<!-- @end-section -->

<!-- @section: evidence -->
## 证据

### 1. 代码路径确实在发 `cache_control`

```
internal/cognitive/core.go:162-169     stable prefix = catalog + durable_memory + context_files
internal/cognitive/core.go:194         BuiltContext.StablePrefixLen
internal/runtime/runtime.go:1074       → 返回给 runtime
internal/runtime/messages.go:46        → pinCachePrefix → messages[0].StablePrefixLen
internal/adapter/http_maas.go:353      → if c.enablePromptCache { stablePrefixLen = req.StablePrefixLen }
internal/adapter/http_maas.go:485-489  → contentPart{ CacheControl: &cacheControl{Type: "ephemeral"} }
```

### 2. 配置的后端没有 Anthropic

`agent.json` 的 MaaS profile：

| profile | model | base_url |
|---|---|---|
| dev / fast / review | `deepseek-v4-flash` | `https://api.deepseek.com` |
| kimik3 | `kimi-k3` | `https://api.kimi.com/coding/v1` |

### 3. DeepSeek 官方语义与 Anthropic 不同

| 维度 | Anthropic | **DeepSeek（Legion 实际后端）** |
|---|---|---|
| 触发方式 | 显式 `cache_control` 断点 | **全自动，服务端，无需任何请求字段** |
| `cache_control` 参数 | 必需 | **不存在** |
| 命中规则 | 断点处前缀精确匹配，**全或无** | **从第 0 token 起的 token 级前缀匹配**（跨 message；实测见 §实测） |
| 中间部分匹配 | 不适用 | **不触发**——且实测表明部分头部匹配**完全不计入**，不是按比例折算 |
| 最小缓存单位 | 512–4096 tokens（因模型而异） | **64 tokens** |
| 上报字段 | `usage.cache_read_input_tokens` | `usage.prompt_cache_hit_tokens` |
| 断点数上限 | 4 | 不适用 |
| 写入成本 | 1.25x（5m TTL） | 无单独写入费 |

来源见文末。

<!-- @end-section -->

<!-- @section: impact -->
## 影响

| # | 影响 | 严重度 |
|---|---|---|
| 1 | `cache_control` 字段无效。**实测已确认：被接受并忽略，HTTP 200**，不会报错——是无害的死代码 | 低（已实测） |
| 2 | `StablePrefixLen` 这条从 `cognitive` 贯穿到 `adapter` 的机制，在当前后端上不产生任何效果 | 中 |
| 3 | **缓存语义的心智模型是错的**，会导出方向相反的优化决策（见下） | **高** |
| 4 | ~~`prompt_cache_hit_tokens` 未被读取或记录~~ — **复核后发现早已接线**（见 §修正方向）。实测基线：稳定 97–98% 命中 | 已解决 |

### 影响 3 的具体表现

本地渲染实测：向能力目录加入 1 个插件工具，stable prefix 由 401 → 458 runes，不再 byte-identical。插件 group 名决定它插在哪里，因而决定与原前缀的公共部分：

| 插件 group 名 | 与原前缀的公共部分 | Anthropic 断点语义下 | **DeepSeek 实测** |
|---|---|---|---|
| `aaa-plugin` | 6.7% | 全失效 | **0 命中** |
| `exec`（插进已有组） | 25.7% | 全失效 | **0 命中** |
| `plugin:jira` | 60.8% | 全失效 | **0 命中** |
| **`zzz-plugin`（排在全部内建之后）** | **77.1%** | **全失效** | **全部命中** |

> ⚠️ **一处推断被实测推翻。** 本文档 v1 曾据「最长公共前缀」推断中间部分命中会按比例折算（60.8% 共同前缀 → 命中 60.8%）。**实测证明不是这样：只要变化点不在末尾，命中就是 0**，共享多少头部都不计入（§实测 Q2）。所以真实对比不是「多保住一些 vs 少保住一些」，而是 **全保 vs 全丢**。

- **断点语义**下，排序对命中率的影响是 0，唯一有效的缓解是拆多个断点或推迟生效。
- **实测语义**下，把变化点移到所有内建条目之后是**唯一**有效的缓解，且收益是全有全无的。

顺带确认一条正面行为：卸载插件后前缀 **byte-identical 恢复**，所以 load/unload 不会造成永久劣化。

<!-- @end-section -->

<!-- @section: fix -->
## 修正方向

### 立即（认知层面）

**先确认后端，再谈缓存语义。** 看到 `cache_control` 就假设后端是 Anthropic，是本次连续误判的根因。任何涉及缓存的判断，第一步是查 `agent.json` 的实际 profile。

### 短期（代码层面）

1. ~~实测 `cache_control` 在 DeepSeek 上的行为~~ — **已完成，见 §实测**：被接受并忽略，不报错。
2. ~~记录 `prompt_cache_hit_tokens`~~ — **复核后发现早已接线**（此前判断有误）：`cachedTokens()` 同时认扁平与 OpenAI 嵌套两种约定 → `resp.CachedTokens` → `st.cachedTokens` 累加 → `audit_events` / `conversation_turns` 的 `cached_tokens` 列。线上真实命中率查库即可。
3. ~~决定 `StablePrefixLen` 的去留~~ — **已定：保留**。复核后发现它本来就是 per-profile 显式 opt-in（`maas.profiles.*.prompt_cache`，默认 `false`），并非无人知道是否生效的中间态。实测结论已写进 `internal/config/config.go` 该字段的注释，接 Anthropic 兼容后端时可直接启用。

### 正确的缓存优化措施（DeepSeek 语义下）

| 措施 | 说明 |
|---|---|
| **能力目录分区渲染** ✅ | 让插件条目排在所有内建条目之后。**已实现**（`Entry.Origin` 一级排序键）——计划见 `legionAgent/docs/superpowers/plans/2026-08-16-capability-render-partition.md`，PR [#79](https://github.com/jxncyjq/stardust-agent-server/pull/79)。实测收益：尾部追加全命中 vs 中间插入 0 命中 |
| **插件变更只在任务边界生效** | 进行中的任务沿用旧目录，避免一次热加载打掉整轮缓存 |
| **保持前缀内一切确定性** | 现有 `Catalog.Entries` 的 (group, name) 排序与 `Render` 的「不加计数/时间戳/id」约束都是对的，继续守住 |
| ~~拆多个 `cache_control` 断点~~ | **不适用**——DeepSeek 没有断点概念 |

<!-- @end-section -->

<!-- @section: measurement -->
## 实测

用 `agent.json` 的 `dev` profile（`deepseek-v4-flash` @ `api.deepseek.com`）发真实请求，四轮探针。system 为约 2000 token 的合成能力目录，`max_tokens=16`（只关心输入侧 usage）。

### Q1 `cache_control` 被拒绝还是被忽略？

**被接受并忽略。** 带该字段与不带的请求都返回 HTTP 200，usage 无差异。它是**无害的死代码**，不会让请求失败。

### Q2 匹配粒度：token 级还是 message 级？

**token 级，且跨 message。** 保持 `system` 逐字节不变、只改 `user` 消息，命中仍约等于 system 长度（1920 tokens）——说明匹配不以 message 为单位。

### Q3 变化点在尾部 vs 中部（决定性）

只有**首次发送**能说明问题：重复发送命中的是它自己刚写入的条目，与放置位置无关。两组独立 salt，各自 warm 基线后测一次首发：

```
baseline 1st (write)    prompt=2093  hit=0      (0%)
baseline 2nd (verify)   prompt=2093  hit=2048   (98%)

TAIL  FIRST SEND        prompt=2116  hit=2048   (97%)   ← 插件追加在尾
MID   FIRST SEND        prompt=2116  hit=0      (0%)    ← 插件插在中间

TAIL first-send hits : [2048, 2048]
MID  first-send hits : [0, 0]
```

**中间插入不是「损失一半」，是损失全部。** MID 变体与基线共享前 30 条（约 1000 tokens），按「最长公共前缀」本应命中一半，实测为 **0**——部分头部匹配完全不计入。

### 结论

| 项 | 结论 |
|---|---|
| `cache_control` | 接受并忽略，HTTP 200，无害死代码 |
| `StablePrefixLen` 链路 | 在当前后端上确实不产生任何效果 |
| 缓存本身 | 工作良好，稳定 **97–98%** 命中 |
| 分区渲染的收益 | **2048 tokens 全命中 vs 全作废**，按 v4-flash 定价（miss `$0.14`/1M，hit `$0.0028`/1M）差异是每次插件变更后第一轮的全价重算 |

### 复现

探针脚本 `probe.py` … `probe4.py`（读 `agent.json` 取 key，不打印）。**只有 `probe4.py` 的口径是对的**——它只测首发；`probe3.py` 的 summary 用了 best-of-two，把「重复发送命中自己」也算进对比，得出「两者相同」的错误结论。

### 已知抖动

第一轮曾出现 TAIL 首发 hit=0，与后续 4 次首发测量不一致。DeepSeek 明说缓存是 **best-effort，不保证 100% 命中**——单次测量不可作为结论，本文的数字取自 2 组独立复现。

<!-- @end-section -->

<!-- @section: kimi -->
## Kimi 实测（2026-08-17）

同一套口径（只测首发、每组独立 salt）跑 `kimik3` profile（`api.kimi.com/coding/v1`，`kimi-k3`），**两次独立复现逐 token 一致**。

```
baseline 1st (write)    prompt=1981  hit=0
baseline 2nd (verify)   prompt=1981  hit=1981   field=prompt_tokens_details.cached_tokens
TAIL  FIRST SEND        prompt=2003  hit=1792
MID   FIRST SEND        prompt=2004  hit=768
cache_control 1st       prompt=1981  hit=0      (HTTP 200，被接受)
cache_control 2nd       prompt=1981  hit=1981
```

### 与 DeepSeek 的差异

| 项 | DeepSeek | Kimi |
|---|---|---|
| usage 字段 | `prompt_cache_hit_tokens`（扁平） | `prompt_tokens_details.cached_tokens`（OpenAI 约定） |
| 尾部追加首发 | 2048 / 2093（98%） | 1792 / 2003（89%） |
| **中段插入首发** | **0**——部分前缀不计分 | **768**——**部分前缀计分** |
| `cache_control` | 接受并忽略 | 接受（HTTP 200），但自动缓存本就全命中，**无法据此证明它起作用** |

> ⚠️ 「两个后端都不吃 `cache_control`、语义相同」这个此前的推断**只对了一半**。Kimi 确实也走自动前缀缓存，但它**给部分头部匹配计分**——正是 DeepSeek 被实测否定的那个行为。跨后端不能互推缓存语义，每个后端都得自己测。

### 对分区渲染（PR #79）的影响

两个后端都有收益，量级不同：DeepSeek 是**全有全无**（2048 vs 0），Kimi 是**部分损失**（1792 vs 768 = 1024 tokens）。改动方向在两者下都成立。

### 代码侧

`adapter/http_maas.go` 的 `cachedTokens()` 已同时覆盖两种约定（先读 `prompt_tokens_details.cached_tokens`，再退 `prompt_cache_hit_tokens`），**无需改动**。

<!-- @end-section -->

<!-- @section: unverified -->
## 仍未验证

| 项 | 方法 | 状态 |
|---|---|---|
| Kimi 是否另有显式 cache 对象 API | 查 Moonshot 文档 | **未查**——自动前缀缓存已实测生效，显式 API 是否另外存在不影响当前结论 |
| MaaS 网关是否会改写/透传该字段 | 抓包或看网关实现 | **未测**（`maas.base_url` 为空，profile 直连各厂商，故本次实测等价于直连行为） |
| `StablePrefixLen` 链路的去留 | 产品决策 | **已定：保留**——它是 per-profile 显式 opt-in（`maas.profiles.*.prompt_cache`），默认 `false`，不是悬空中间态；实测结论已写进 `internal/config/config.go` 的字段注释，接 Anthropic 兼容后端时可直接启用 |
| 生产环境真实命中率 | 落盘缓存命中 token | **已接线**（此前判断有误）——`cachedTokens()` → `resp.CachedTokens` → `st.cachedTokens` 累加 → `audit_events` / `conversation_turns` 的 `cached_tokens` 列。实测的 97–98% 仍来自合成前缀，线上真实值需查库 |

<!-- @end-section -->

## 参考来源

- [Context Caching | DeepSeek API Docs](https://api-docs.deepseek.com/guides/kv_cache/)
- [DeepSeek API introduces Context Caching on Disk](https://api-docs.deepseek.com/news/news0802/)
- [DeepSeek Context Caching: How It Works, Cache Hits, and API Cost Optimization](https://deepseek-usa.ai/docs/deepseek-context-caching/)
- [DeepSeek API Context Caching: ~98% Cheaper Cached Input on V4](https://deepseekai.guide/api/deepseek-api-context-caching/)

## 相关文档

- [[design-legion-plugin-system-001|Legion 插件系统设计方案]] — 插件动态增删是本问题的触发场景
- [[bug-explore-hallucinated-paths-001|BUG — 探索项目时模型臆想 monorepo 路径]] — 同属 agent 行为/成本取证类
