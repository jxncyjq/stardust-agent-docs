---
id: "spec-component-registry-impl-000"
title: "ComponentRegistry 注册与管理规范"
aliases: ["ComponentRegistry规范", "组件注册管理", "component-registry-impl-spec"]
type: "spec"
category: "design/architecture/components"
tags: ["component-spec", "registry", "lifecycle", "dependency-injection", "framework"]
version: "0.1.0"
created: "2026-05-09"
updated: "2026-05-09"
author: "jxncyjq"
status: "draft"
parent: "spec-component-registry-000"

component_id: "C00"
layer: "L0"
depends_on: []
optional_deps: []
conflicts_with: []
required_by:
  - "ALL"   # 所有组件都通过 Registry 注册和发现
assembly_profiles:
  - minimal
  - standard
  - enterprise
  - embedded
---

<!-- @section: overview -->
# ComponentRegistry 注册与管理规范

## 1. 组件定位

`ComponentRegistry` 是框架的**组件注册中心和依赖容器**，是所有其他组件运转的基础设施。它负责：

1. **注册**：接收组件实例，按 `ComponentID` 索引
2. **依赖校验**：启动时拓扑排序，检测循环依赖、缺失必须依赖、互斥冲突
3. **Noop 注入**：为缺失的可选依赖注入对应的 Noop 实现
4. **生命周期管理**：按拓扑顺序调用 `SelfCheck` / `Shutdown`
5. **Profile 装配**：按预设方案加载组件并应用覆盖

它本身**不持有业务逻辑**，仅负责装配和编排。其他组件通过 `Registry.Get(ID)` 访问对端依赖。

```
应用启动入口
   │
   ├─ 1. NewRegistry()               // 创建空容器
   ├─ 2. UseProfile(ProfileStandard) // 声明需要的组件集
   ├─ 3. Register(实例1, 实例2, ...)  // 注入实现
   ├─ 4. Build(ctx)                  // 校验 + Noop 注入 + SelfCheck
   │      │
   │      ├─ 拓扑排序          ┐
   │      ├─ 校验依赖完整性    ├─ 失败 → panic / 返回 error
   │      ├─ 注入 Noop         ┘
   │      ├─ L0 → ... → L7 SelfCheck
   │      └─ 标记 Ready
   │
   └─ 5. Serve(...)                  // 接受请求
```

**与 [[component-registry|组件注册表与依赖关系图]] 的关系**：
- `component-registry.md` 是**主表/目录文档**：定义 ID、依赖关系、Profile、Noop 行为表
- 本文档是 **C00 组件本身的代码规范**：定义 `Registry` Go 接口、注册 API、校验算法、契约测试

<!-- @end-section -->

<!-- @section: interface -->
---

## 2. 接口定义

### 2.1 核心接口

```go
// Registry 是组件注册中心。
// 实现方需要并发安全：注册阶段可被多个 goroutine 调用，Build 后变为只读。
type Registry interface {
    // UseProfile 声明使用某个预设装配方案。
    // 多次调用合并 Required/Optional 集合（Conflicts 取并集）。
    // 必须在 Register 之前调用。
    UseProfile(profile AssemblyProfile) Registry

    // Register 注册一个组件实例。
    // 同一 ComponentID 重复注册返回错误（除非显式 Override）。
    // Build 之后调用返回 ErrRegistryFrozen。
    Register(c Component) error

    // MustRegister 注册失败时 panic（用于 main 入口的便捷写法）。
    MustRegister(c Component)

    // Override 替换已注册的组件（用于测试或灰度）。
    // Build 之后调用返回 ErrRegistryFrozen。
    Override(c Component) error

    // Get 按 ID 获取组件，未注册时返回 (nil, false)。
    // Build 之前调用返回 ErrRegistryNotBuilt。
    Get(id ComponentID) (Component, bool)

    // MustGet 未注册时 panic（仅在能确定组件存在时使用，例如 Profile.Required）。
    MustGet(id ComponentID) Component

    // Build 校验依赖、注入 Noop、调用 SelfCheck。
    // 校验失败返回 *BuildError，包含完整的错误清单。
    // 同一 Registry 只能 Build 一次。
    Build(ctx context.Context) (*Framework, error)

    // Snapshot 返回当前注册状态的只读视图（用于诊断 / /debug 端点）。
    Snapshot() RegistrySnapshot
}
```

### 2.2 Component 与生命周期接口

```go
// Component 所有可注册组件的最小公共接口。
type Component interface {
    ID() ComponentID
}

// ComponentLifecycle 框架感知接口（可选实现，按需断言）。
type ComponentLifecycle interface {
    // SelfCheck 在 Build 末尾调用，按拓扑顺序执行（依赖先于被依赖者）。
    // 返回 error 阻止框架启动；ctx 含超时（默认 10s/组件）。
    SelfCheck(ctx context.Context) error

    // Shutdown 在 Framework.Stop() 时调用，按反向拓扑顺序执行。
    // ctx 含超时（默认 30s/组件），超时后被强制中断。
    Shutdown(ctx context.Context) error
}

// DependencyAware 组件可声明运行时依赖（用于 Inject 模式，见 §3.2）。
// 大多数组件通过构造函数注入依赖；只有少数需要在注册后再访问 Registry 时实现此接口。
type DependencyAware interface {
    // Inject 在所有组件注册完成后、SelfCheck 之前调用。
    // 实现方在此从 reg 中取出依赖并保存到自身字段。
    Inject(reg Registry) error
}

// ComponentID 类型化的组件标识。
type ComponentID string
```

### 2.3 装配方案

```go
// AssemblyProfile 描述一套组件组合。
type AssemblyProfile struct {
    Name        string
    Description string
    Required    []ComponentID  // 必须注册（否则 Build 失败）
    Optional    []ComponentID  // 可选（缺失时注入 Noop）
    Conflicts   []ComponentID  // 禁用（同时注册则 Build 失败）
}

// 内置 Profile 常量（在 component-registry.md 中详细描述）。
var (
    ProfileMinimal    AssemblyProfile
    ProfileStandard   AssemblyProfile
    ProfileEnterprise AssemblyProfile
    ProfileEmbedded   AssemblyProfile
)
```

### 2.4 Build 输出

```go
// Framework 是 Registry.Build 成功后的产物。
// 提供运行时入口（HTTP、gRPC、SDK）。
type Framework struct {
    registry Registry
    // 生命周期钩子（已注册顺序）
    lifecycles []ComponentLifecycle
}

func (f *Framework) Registry() Registry
func (f *Framework) Serve(addr string) error
func (f *Framework) Stop(ctx context.Context) error  // 按反向拓扑调 Shutdown
```

<!-- @end-section -->

<!-- @section: dependency-validation -->
---

## 3. 依赖校验算法

### 3.1 拓扑排序与校验流程

```
Build(ctx) 内部步骤：
1. 收集所有已注册组件的 Front Matter 依赖元数据
   （depends_on / optional_deps / conflicts_with 来自组件类型注册时的元信息）

2. 检查 Profile 约束：
   - 所有 profile.Required 必须已 Register
   - 所有 profile.Conflicts 必须未 Register
   - 同时注册了互相 conflicts_with 的组件 → 失败

3. 对所有已注册组件构建依赖图（仅 depends_on 边）
   Kahn 算法拓扑排序：
   - 计算入度
   - 入度 0 的入队
   - 出队 → 递减下游入度 → 入度 0 的入队
   - 队列为空时若仍有未排序节点 → 循环依赖

4. 对每个组件验证 depends_on 全部已注册（否则报缺失）

5. 对 optional_deps 中未注册的依赖：
   - 查表注入对应 Noop 实现（见 §4）
   - 已注册则跳过

6. 调用所有实现 DependencyAware 的组件的 Inject(reg)

7. 按拓扑顺序对实现 ComponentLifecycle 的组件调用 SelfCheck
   - 任一返回 error → 终止启动，已 SelfCheck 的组件按反向顺序调 Shutdown 回滚

8. 返回 *Framework
```

### 3.2 依赖元数据来源

每个组件类型在初始化期向 Registry **类型注册表**登记自身依赖（一次性，与实例无关）：

```go
// RegisterComponentType 在 init() 阶段为某个 ComponentID 登记依赖元信息。
// 由各组件包的 init() 调用，与该组件的实例创建解耦。
func RegisterComponentType(meta ComponentMeta)

type ComponentMeta struct {
    ID           ComponentID
    Layer        Layer            // L0 ~ L7
    DependsOn    []ComponentID
    OptionalDeps []ComponentID
    ConflictsWith []ComponentID
    NoopFactory  func() Component // 可选；optional_deps 缺失时使用
}
```

> 元信息源自规范文档 Front Matter，由代码生成器或手工保持同步。

### 3.3 错误聚合

```go
// BuildError 聚合 Build 阶段的所有错误，一次性返回（避免一个个提示）。
type BuildError struct {
    Missing    []MissingDep   // depends_on 缺失
    Conflicts  []ConflictPair // 互斥冲突
    Cycles     [][]ComponentID // 循环依赖（每个元素是一个环）
    SelfCheck  []SelfCheckErr // SelfCheck 失败
}

func (e *BuildError) Error() string  // 多行可读输出，含修复建议

type MissingDep struct {
    Component ComponentID
    Missing   ComponentID
    Suggest   string  // 例 "go get github.com/x/y; reg.Register(y.New())"
}
```

<!-- @end-section -->

<!-- @section: noop-injection -->
---

## 4. Noop 注入机制

### 4.1 Noop 适用对象

只对 `optional_deps` 中**缺失**的组件注入 Noop。`depends_on` 缺失会直接 Build 失败。

### 4.2 Noop 来源

每个支持 Noop 的组件在类型注册时提供 `NoopFactory`：

```go
// 例：在 trace-collector 的 init 中
func init() {
    component.RegisterComponentType(component.ComponentMeta{
        ID:          "C60",
        Layer:       component.LayerL7,
        NoopFactory: func() component.Component { return NewNoopTraceCollector() },
    })
}
```

### 4.3 Noop 行为统一表

详见 [[component-registry#5-3-noop-实现规则|component-registry §5.3]]。Noop 实现必须满足：

| 要求 | 说明 |
|------|------|
| **同实例兼容** | Noop 实现完整接口，调用方无需类型判断 |
| **零副作用** | 不写日志、不发起网络、不持有资源 |
| **零成本** | 所有方法应是 O(1) inline 级别 |
| **可识别** | `Component.ID()` 返回原 ID（不带 `-noop` 后缀，调用方无感知）|

<!-- @end-section -->

<!-- @section: registration-api -->
---

## 5. 注册 API

### 5.1 程序化注册

```go
func main() {
    reg := component.NewRegistry().
        UseProfile(component.ProfileStandard)

    // 必填依赖
    reg.MustRegister(provider.NewthemodelProvider(cfg.themodel))
    reg.MustRegister(provider.NewProviderRegistry())  // 内部从 reg 取所有 ModelProvider
    reg.MustRegister(billing.NewPricingEngine(cfg.Pricing))
    reg.MustRegister(billing.NewWalletFundingSource(db))
    reg.MustRegister(billing.NewBillingSession())
    reg.MustRegister(reliability.NewExponentialBackoffScheduler())
    reg.MustRegister(reliability.NewFailoverManager())
    reg.MustRegister(stream.NewStreamProxy())

    // 可选依赖（覆盖 Noop）
    if cfg.OTLP.Enabled {
        reg.MustRegister(observe.NewOTLPTraceCollector(cfg.OTLP))
    }
    reg.MustRegister(observe.NewPrometheusMetricsRecorder())

    fw, err := reg.Build(context.Background())
    if err != nil {
        log.Fatalf("framework build failed:\n%s", err)
    }
    log.Fatal(fw.Serve(":8080"))
}
```

### 5.2 配置驱动注册

框架提供 `LoadFromConfig`，根据 YAML 配置实例化对应组件：

```go
func LoadFromConfig(path string) (Registry, error)
```

YAML 示例见 [[component-registry#6-2-配置文件注册-yaml|component-registry §6.2]]。

### 5.3 测试场景：Override

```go
func TestSomething(t *testing.T) {
    reg := component.NewTestRegistry().
        UseProfile(component.ProfileMinimal)

    // 注入 Mock
    reg.MustRegister(&MockModelProvider{IDValue: "test-provider"})
    reg.MustRegister(&MockFundingSource{})
    // ...

    fw, err := reg.Build(t.Context())
    require.NoError(t, err)
    defer fw.Stop(context.Background())

    // ...
}

// 替换某个组件用于灰度测试
reg.Override(experimentalRouter)
```

<!-- @end-section -->

<!-- @section: lifecycle -->
---

## 6. 生命周期与并发模型

### 6.1 状态机

```
[空] ──Register/Override──► [注册中] ──Build──► [运行中] ──Stop──► [已关闭]
        │                       │                  │
        │                       │                  └── Shutdown 反向拓扑
        │                       └── 校验失败 → [失败]（不可恢复）
        └── Build 之前可重复 Register
```

### 6.2 并发安全

| 阶段 | 操作 | 并发性 |
|------|------|--------|
| 注册中 | `Register` / `Override` | 互斥锁保护，可被多 goroutine 调用 |
| 注册中 | `Get` | 返回 `ErrRegistryNotBuilt`（避免读取未完成视图）|
| 运行中 | `Get` / `MustGet` / `Snapshot` | 无锁（注册映射 Build 后冻结）|
| 运行中 | `Register` / `Override` | 返回 `ErrRegistryFrozen` |
| 关闭中 | 任何方法 | 返回 `ErrRegistryClosed` |

### 6.3 SelfCheck 与 Shutdown 顺序

```
SelfCheck 顺序（依赖先行，与初始化一致）：
  L0 → L1 → L2 → L3 → L4 → L5 → L6 → L7
  同一层内按 Register 顺序

Shutdown 顺序（反向，确保依赖方先停）：
  L7 → L6 → L5 → L4 → L3 → L2 → L1 → L0
  同一层内按 Register 反序

每个组件的 SelfCheck/Shutdown 单独有超时（默认 10s/30s）。
任一 SelfCheck 失败：
  - 已成功 SelfCheck 的组件按反向顺序调 Shutdown 回滚
  - Build 返回 *BuildError
```

<!-- @end-section -->

<!-- @section: errors -->
---

## 7. 错误类型

```go
var (
    // ErrAlreadyRegistered Register 时同 ID 已存在（且未使用 Override）。
    ErrAlreadyRegistered = errors.New("component already registered")

    // ErrRegistryFrozen Build 之后再调用 Register/Override。
    ErrRegistryFrozen = errors.New("registry is frozen after Build")

    // ErrRegistryNotBuilt Build 之前调用 Get 等只读方法。
    ErrRegistryNotBuilt = errors.New("registry has not been built")

    // ErrRegistryClosed Stop 之后任何方法。
    ErrRegistryClosed = errors.New("registry is closed")
)
```

`BuildError` 的字段已在 §3.3 描述。

<!-- @end-section -->

<!-- @section: behavior-contracts -->
---

## 8. 行为契约

| 契约 | 说明 |
|------|------|
| **Build 一次性** | 同一 Registry 实例只能成功 Build 一次；失败后不可重试 |
| **Profile 合并** | `UseProfile` 多次调用：Required/Optional 取并集，Conflicts 取并集 |
| **Override 不绕过校验** | Override 后的组件依然参与依赖图校验 |
| **Noop 透明** | 调用方代码不应区分 Noop 与真实实现 |
| **元信息单一来源** | `ComponentMeta` 通过 `RegisterComponentType` 在 init 期登记，不在运行时变更 |
| **生命周期反序关闭** | Shutdown 严格按拓扑反向；同层按注册反序 |
| **失败聚合** | Build 校验失败时一次性报告所有问题（不在第一个错误处中断） |
| **测试 Registry 隔离** | `NewTestRegistry()` 与生产 Registry 共享类型元数据，但实例集合独立 |

<!-- @end-section -->

<!-- @section: testing -->
---

## 9. 契约测试

```go
// RunRegistryContractTests 验证任何 Registry 实现遵守接口契约。
func RunRegistryContractTests(t *testing.T, factory func() Registry) {
    t.Run("Register then Get returns same instance", func(t *testing.T) {
        reg := factory()
        c := &mockComp{id: "M1"}
        require.NoError(t, reg.Register(c))
        _, err := reg.Build(t.Context())
        require.NoError(t, err)
        got, ok := reg.Get("M1")
        require.True(t, ok)
        require.Same(t, c, got)
    })

    t.Run("duplicate Register returns ErrAlreadyRegistered", func(t *testing.T) {
        reg := factory()
        require.NoError(t, reg.Register(&mockComp{id: "M1"}))
        err := reg.Register(&mockComp{id: "M1"})
        require.ErrorIs(t, err, ErrAlreadyRegistered)
    })

    t.Run("Build twice returns ErrRegistryFrozen on second", func(t *testing.T) {
        reg := factory()
        _, err := reg.Build(t.Context())
        require.NoError(t, err)
        err = reg.Register(&mockComp{id: "M1"})
        require.ErrorIs(t, err, ErrRegistryFrozen)
    })

    t.Run("missing required dep aggregates in BuildError", func(t *testing.T) {
        reg := factory().UseProfile(ProfileStandard)
        _, err := reg.Build(t.Context())
        var be *BuildError
        require.ErrorAs(t, err, &be)
        require.NotEmpty(t, be.Missing)
    })

    t.Run("optional dep auto-injects Noop", func(t *testing.T) {
        reg := factory()
        // 注册一个声明 optional_deps=[C60] 的组件，但不注册 C60
        reg.MustRegister(newComponentNeedingTraceCollector())
        _, err := reg.Build(t.Context())
        require.NoError(t, err)
        tc, ok := reg.Get("C60")
        require.True(t, ok)
        require.IsType(t, &NoopTraceCollector{}, tc)
    })

    t.Run("cyclic dependency reported", func(t *testing.T) {
        // 注入两个虚假组件，依赖互相指向
        // 期望 BuildError.Cycles 非空
    })

    t.Run("SelfCheck failure triggers reverse Shutdown of completed", func(t *testing.T) {
        // 注册三个组件，第三个 SelfCheck 返回 error
        // 验证前两个的 Shutdown 被调用且顺序正确
    })

    t.Run("Shutdown order is reverse topology", func(t *testing.T) {
        // 验证 L7→L0、同层注册反序
    })
}
```

### Mock 与测试 Helper

```go
// NewTestRegistry 返回不强制 Profile 的 Registry，便于单测。
func NewTestRegistry() Registry

// mockComp 测试用最小组件。
type mockComp struct {
    id          ComponentID
    selfCheckFn func(context.Context) error
    shutdownFn  func(context.Context) error
}

func (m *mockComp) ID() ComponentID                       { return m.id }
func (m *mockComp) SelfCheck(ctx context.Context) error   { return safe(m.selfCheckFn, ctx) }
func (m *mockComp) Shutdown(ctx context.Context) error    { return safe(m.shutdownFn, ctx) }
```

<!-- @end-section -->

<!-- @section: checklist -->
---

## 10. 实现检查清单

```
Registry 接口
  ☐ NewRegistry / NewTestRegistry
  ☐ UseProfile（合并语义）
  ☐ Register / MustRegister / Override
  ☐ Get / MustGet
  ☐ Build：原子（要么完全成功要么完全失败）
  ☐ Snapshot：只读视图

依赖校验
  ☐ Kahn 拓扑排序
  ☐ 循环依赖检测（输出所有环）
  ☐ depends_on 缺失检测
  ☐ optional_deps Noop 注入
  ☐ conflicts_with 互斥检测
  ☐ Profile.Required/Conflicts 校验
  ☐ 错误一次性聚合到 BuildError

类型元信息
  ☐ RegisterComponentType（init 阶段调用）
  ☐ ComponentMeta：DependsOn/OptionalDeps/ConflictsWith/NoopFactory
  ☐ 元数据生成器（从 spec front matter 同步，可选）

生命周期
  ☐ DependencyAware.Inject 在 Build 期间调用
  ☐ SelfCheck 按拓扑顺序、单组件超时
  ☐ Shutdown 按反向拓扑、单组件超时
  ☐ SelfCheck 失败回滚已成功部分

并发安全
  ☐ 注册阶段互斥锁
  ☐ Build 后只读访问无锁
  ☐ 状态机转换原子

错误处理
  ☐ ErrAlreadyRegistered / ErrRegistryFrozen / ErrRegistryNotBuilt / ErrRegistryClosed
  ☐ BuildError.Error() 多行可读，含修复建议

测试
  ☐ 通过 RunRegistryContractTests
  ☐ 至少测试每种 BuildError 字段
  ☐ Shutdown 反向顺序的单测
```

<!-- @end-section -->

<!-- @section: related -->
---

## 相关文档

- [[component-registry|组件注册表与依赖关系图]]（C00 主表，依赖关系/Profile/Noop 行为权威来源）
- [[model-provider-spec|ModelProvider 组件规范]]（C01 通过 Registry 注册示例）
- 各组件 spec 的 front matter `depends_on` / `optional_deps` 字段（元信息来源）

<!-- @end-section -->
