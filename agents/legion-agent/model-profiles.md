---
id: "reference-maas-model-profiles-001"
title: "MaaS Model Profiles 参考手册"
aliases: ["model profiles", "maas profiles", "profile 选择", "指定 profile"]
type: "reference"
category: "agents/legion-agent"
tags: ["maas", "profile", "configuration", "cli", "tui"]
version: "1.1.0"
created: "2026-04-26"
updated: "2026-05-18"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "reference-configuration-001"
    relation: "related_to"
    path: "./configuration.md"
---

# MaaS Model Profiles 参考手册

<!-- @section: overview -->
## 概述

Legion Agent 支持多模型 profile 配置。通过 profile，可以在同一份 `agent.json` 中声明多个 MaaS 后端（如 DeepSeek、OpenAI-compatible 接口、本地代理等），并在启动时选择使用哪个 profile，而无需修改配置文件本身。
<!-- @end-section -->

<!-- @section: profile-selection -->
## 启动时如何指定 Profile

Profile 选择遵循以下优先级，**高优先级覆盖低优先级**：

| 优先级 | 方式 | 示例 |
|--------|------|------|
| 1（最高）| CLI flag `--maas-url` | `--maas-url https://... --maas-api-key sk-xxx`，直接绑定裸 URL，**完全绕过 profile 系统** |
| 2 | CLI flag `--maas-profile` | `--maas-profile deepseek`，选择 `maas.profiles` 中的具名 profile |
| 3 | 配置文件 `maas.default_profile` | 未传 flag 时自动使用 `agent.json` 中声明的默认 profile |
| 4 | 顶层 `maas.base_url` | 无 profile 时回退到旧版单端点配置 |
| 5（最低）| 无任何 MaaS 配置 | 回落为内置 demo 响应（`runtime.demo_response` 字段） |

### 用 `--maas-profile` Flag

```powershell
# run 命令
agent run --config agent.json --maas-profile deepseek --prompt "Summarize this"

# tui 命令
agent tui --config agent.json --maas-profile gpt4o
```

Flag 值必须与 `maas.profiles` 中的 key 完全匹配（大小写敏感）。

### 用 `default_profile` 配置默认值

```json
{
  "maas": {
    "default_profile": "deepseek",
    "profiles": {
      "deepseek": { ... },
      "gpt4o":    { ... }
    }
  }
}
```

不传 `--maas-profile` 时，agent 自动选用 `deepseek` profile。

### 临时用 `--maas-url` 覆盖

```powershell
# 忽略所有 profile，直接打到指定 URL
agent tui --config agent.json --maas-url http://127.0.0.1:11434 --maas-api-key ""
```

> ⚠️ `--maas-url` 一旦传入，`--maas-profile` 和 `default_profile` 均被忽略。
<!-- @end-section -->

<!-- @section: configuration -->
## Profile 配置格式

```json
{
  "maas": {
    "default_profile": "dev",
    "profiles": {
      "dev": {
        "base_url": "https://api.deepseek.com",
        "model":    "deepseek-chat",
        "api_key":  "sk-xxx"
      },
      "fast": {
        "base_url": "https://maas-fast.example.test",
        "model":    "",
        "api_key":  "fast-key"
      }
    }
  }
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `base_url` | string | 是 | MaaS 服务地址 |
| `model` | string | 否 | 非空时使用 OpenAI-compatible `/chat/completions`；为空时使用 Legion 内部 `/v1/inference/generate` |
| `api_key` | string | 否 | Bearer token；不填则不发送 Authorization header |

### 两种协议

- **`model` 非空** → `POST {base_url}/chat/completions`，请求体包含 `model` + `messages`（OpenAI-compatible）
- **`model` 为空** → `POST {base_url}/v1/inference/generate`（Legion 内部 MaaS 协议）

### 旧版单端点（兜底）

无需迁移，旧配置仍可用：

```json
{
  "maas": {
    "base_url": "https://maas.example.test",
    "api_key":  "legacy-key"
  }
}
```
<!-- @end-section -->

<!-- @section: tui-display -->
## TUI 中的 Profile 显示

TUI 标题栏会显示当前使用的 agent 名和 model 名：

- 使用具名 profile 时：显示 profile key 作为 agent name，`profile.model` 作为 model name
- 使用 `--maas-url` 时：显示 `agent · custom-maas`
- 使用旧版 `maas.base_url` 时：显示 `agent · maas`
- 无任何 MaaS 配置时：显示 `agent · recording`
<!-- @end-section -->

<!-- @section: env-override -->
## 环境变量

| 环境变量 | 等效于 |
|----------|--------|
| `LEGION_AGENT_MAAS_URL` | 顶层 `maas.base_url`（不作用于 profile 内部） |
| `LEGION_AGENT_MAAS_API_KEY` | 顶层 `maas.api_key` |

> Profile 内部的 `base_url` / `api_key` 目前不支持环境变量覆盖，需在配置文件中直接修改或通过 `--maas-url` / `--maas-api-key` flag 临时替换。
<!-- @end-section -->

<!-- @section: security -->
## 安全说明

- API key 不会出现在 OpenAPI schema、diagnostics endpoint、日志或 TUI 界面中。
- Profile 名称可能出现在日志和命令行历史中，请勿把 API key 放入 profile 名称。
- 生产环境建议通过 CI/CD secret 注入 `api_key`，在本地用 `--maas-api-key` 临时覆盖，避免明文写入 `agent.json`。
<!-- @end-section -->

## 相关文档

- [[reference-configuration-001|配置参考手册]] — 完整配置字段说明
