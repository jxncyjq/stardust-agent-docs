---
id: "spec-component-model-mapper-004"
title: "ModelMapper 组件规范"
aliases: ["ModelMapper规范", "模型映射器", "model-mapper-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "routing", "alias", "mapping", "maas"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C04"
layer: "L1"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "C14"   # ModelRouter 在路由前通过 ModelMapper 解析模型别名
assembly_profiles:
  - standard
  - enterprise
  - embedded
---

<!-- @section: overview -->
# ModelMapper 组件规范

## 1. 组件定位

`ModelMapper` 负责将用户请求中的**模型名称**（可能是别名、旧版本名、平台统一名）解析为框架内部的**规范模型 ID**，再进一步映射到各提供商侧的实际模型 ID。

它是路由层的前置步骤，使得：
- 用户可以使用 `"gpt-4"` 请求，框架路由到 `openai/gpt-4o`（最新兼容版本）
- 同一框架模型名可以被不同提供商支持（路由层决定选哪个）

```
用户请求: model="claude-3-opus"
        │
        ▼ ModelMapper.Resolve("claude-3-opus")
        │
        ▼ frameworkModelID = "claude-opus-4-7"  （规范名）
        │
        ▼ ProviderRegistry.ListByModel("claude-opus-4-7")
        │
        ▼ [anthropic→"claude-opus-4-7", azure→"claude-opus-4-7-20251101"]
                        ↑ 提供商侧实际模型名（来自 model_capabilities 表）
```

**读者**：配置模型别名的运营人员、集成路由层的开发者。

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

```go
// ModelMapper 将用户侧模型名解析为框架规范模型 ID。
// 并发安全：映射表在启动时加载，运行期只读。
type ModelMapper interface {
    // Resolve 将用户请求的模型名（别名或规范名）解析为框架规范模型 ID。
    // 输入已是规范名时，直接返回（pass-through）。
    // 找不到映射时返回 ErrModelNotFound。
    Resolve(requestedModel string) (frameworkModelID string, err error)

    // ResolveAll 返回所有框架模型 ID 的完整列表（用于 /models 接口）。
    ResolveAll() []ModelInfo

    // Reload 重新从配置源加载映射表（热更新）。
    Reload(ctx context.Context) error
}

// ModelInfo 模型的公开信息（用于 /models 列表接口）。
type ModelInfo struct {
    FrameworkModelID string   // 框架规范 ID，如 "claude-opus-4-7"
    Aliases          []string // 该模型的所有别名，如 ["claude-3-opus", "claude-opus"]
    DisplayName      string   // 展示名称
    Deprecated       bool     // 是否已弃用（仍可用，但建议迁移）
}
```

<!-- @end-section -->

<!-- @section: alias-rules -->
---

## 3. 别名规则

### 3.1 别名类型

| 类型 | 示例 | 说明 |
|------|------|------|
| 规范名 → 自身 | `claude-opus-4-7` | Pass-through，无需映射 |
| 旧版本别名 | `claude-3-opus` → `claude-opus-4-7` | 版本升级兼容 |
| 平台别名 | `gpt-4` → `gpt-4o` | OpenAI 兼容层 |
| 短别名 | `claude-opus` → `claude-opus-4-7` | 方便引用最新版本 |
| 弃用别名 | `text-davinci-003` → `gpt-3.5-turbo` | 仍可用，标记 Deprecated |

### 3.2 映射配置（YAML）

```yaml
model_aliases:
  # 规范名是 key，aliases 是所有指向它的别名
  claude-opus-4-7:
    display_name: "Claude Opus 4.7"
    aliases:
      - claude-3-opus
      - claude-opus
      - claude-3-opus-20240229

  claude-sonnet-4-6:
    display_name: "Claude Sonnet 4.6"
    aliases:
      - claude-sonnet
      - claude-3-5-sonnet
      - claude-3-5-sonnet-20241022

  gpt-4o:
    display_name: "GPT-4o"
    aliases:
      - gpt-4
      - gpt-4-turbo
      - gpt-4-1106-preview
    deprecated_aliases:
      - gpt-4-0314    # 仍可路由，但标记弃用
```

### 3.3 数据库方式

```sql
-- 模型别名表
CREATE TABLE model_aliases (
    id               BIGSERIAL    PRIMARY KEY,
    alias            VARCHAR(128) NOT NULL UNIQUE,
    framework_model  VARCHAR(128) NOT NULL,
    deprecated       BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- 规范名本身也是一条 alias（alias = framework_model），简化查询逻辑
INSERT INTO model_aliases (alias, framework_model) VALUES
    ('claude-opus-4-7', 'claude-opus-4-7'),
    ('claude-3-opus',   'claude-opus-4-7'),
    ('claude-opus',     'claude-opus-4-7');
```

<!-- @end-section -->

<!-- @section: config -->
---

## 4. 配置 Schema

```go
// ModelMapperConfig 是 ModelMapper 的配置。
type ModelMapperConfig struct {
    // Source 别名数据来源。
    Source ModelAliasSource `validate:"required"`

    // StrictMode 严格模式：true = 未知模型名返回错误；false = pass-through（原样返回）。
    // 生产建议 true；开发/调试时可设为 false 方便直接传提供商模型名。
    StrictMode bool `default:"true"`

    // ReloadInterval 自动重新加载的间隔（0 = 禁用）。
    ReloadInterval time.Duration `default:"5m"`
}

type ModelAliasSource struct {
    Type     string // "yaml" | "database"
    FilePath string
    Table    string `default:"model_aliases"`
}
```

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 5. 行为契约

| 契约 | 说明 |
|------|------|
| **Resolve 不做 IO** | 别名表在启动时加载到内存，运行期不查 DB |
| **规范名 pass-through** | 输入已是有效规范名时，直接返回，不报错 |
| **StrictMode=false 的 pass-through** | 未知名称在非严格模式下原样返回（方便直接传提供商模型名） |
| **大小写不敏感（可选配置）** | 默认不忽略大小写；`CaseInsensitive=true` 时统一转小写匹配 |
| **Reload 原子替换** | 热更新期间飞行中的请求使用旧表，新请求使用新表 |
| **别名唯一** | 同一别名不能指向两个不同的框架模型（配置校验时检测） |

<!-- @end-section -->

<!-- @section: error-types -->
---

## 6. 错误类型

```go
var (
    // ErrModelNotFound 请求的模型名在 StrictMode=true 时找不到映射。
    // 调用方（ModelRouter）应返回 400。
    ErrModelNotFound = errors.New("model not found in alias table")

    // ErrAmbiguousAlias 别名冲突（加载时检测，启动失败）。
    ErrAmbiguousAlias = errors.New("alias maps to multiple framework models")
)
```

<!-- @end-section -->

<!-- @section: testing -->
---

## 7. 测试支持

```go
// RunModelMapperContractTests 验证 ModelMapper 实现的行为契约。
func RunModelMapperContractTests(t *testing.T, mapper ModelMapper) {
    t.Run("Resolve/CanonicalName/PassThrough", ...)
    t.Run("Resolve/Alias/ReturnsCanonical", ...)
    t.Run("Resolve/Unknown/StrictMode/ReturnsError", ...)
    t.Run("Resolve/Unknown/NonStrict/PassThrough", ...)
    t.Run("Resolve/ConcurrencySafe", ...)
    t.Run("ResolveAll/ContainsAllCanonicalNames", ...)
}
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 8. 实现检查清单

```
ModelMapper
  ☐ Resolve：别名 → 框架规范名，O(1) 查找（map 实现）
  ☐ Resolve：规范名本身作为别名注册（pass-through）
  ☐ StrictMode=true：未知名称返回 ErrModelNotFound
  ☐ StrictMode=false：未知名称原样返回
  ☐ 不做 IO（运行期纯内存）
  ☐ Reload：原子替换别名表

配置加载
  ☐ 支持 YAML 和数据库两种来源
  ☐ 启动时检测别名冲突（AmbiguousAlias），冲突则拒绝启动

测试
  ☐ 运行 RunModelMapperContractTests（全部通过）
  ☐ 别名冲突检测测试
  ☐ 并发安全验证
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C04 在依赖图中的位置）
- model-router-spec.md（C14，框架在路由前调用 ModelMapper）
- provider-registry-spec.md（C03，使用框架规范名查询候选提供商）

<!-- @end-section -->
