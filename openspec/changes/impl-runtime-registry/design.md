## Context

runtime-registry 是 deeptutor-go Wave 1 / M1 的第一个模块：定义 Level 1 工具与 Level 2 capability 的注册、查询与统一路由（ChatOrchestrator）。基线实现为 `deeptutor/runtime/orchestrator.py`、`deeptutor/runtime/registry/tool_registry.py`、`deeptutor/runtime/registry/capability_registry.py`、`deeptutor/core/tool_protocol.py`、`deeptutor/core/capability_protocol.py`。约束：行为与基线 v1.5.2 对等（本模块无 REST/WS 直接暴露面，对等体现在 schema 生成、事件序、错误文案语义）；技术栈固定为 Go ≥ 1.23、eino（Agent 框架）、goroutine/channel（StreamBus）。上游依赖 `impl-foundation-config`（settings）与 `impl-foundation-stream`（`StreamEvent`/`StreamBus` 类型）。

## Goals / Non-Goals

**Goals:**

- 提供与基线对等的 `ToolRegistry` / `CapabilityRegistry` / `ChatOrchestrator`，接口签名冻结为下游（agent-loop、turn-runtime、tools-builtin）的编译期契约。
- OpenAI function-calling schema 生成逐字段对等（`raw_parameters` 优先、array 缺省 `items` 回退），可直接喂给 eino ChatModel 的 tools 绑定。
- Orchestrator 的事件序（`session` → capability 事件 → `DONE`）与错误转化语义对等。

**Non-Goals:**

- 不实现任何具体工具（tools-builtin）与 capability 流水线（agent-loop / capability-*）；本模块只放接口、注册表与 2~3 个测试用 fake。
- 不实现 MCP 工具接入（deferred 池只留 `Deferred` 标志与查询接口）。
- 不实现全局 EventBus 本体（cron-events 模块）；此处只依赖其发布接口并容忍缺席。

## Decisions

### D1. Go 包结构

```
internal/
├── core/                       # 协议类型（无业务依赖，全仓最底层）
│   ├── tool_protocol.go        # Tool / ToolDefinition / ToolParameter / ToolResult / ToolPromptHints
│   ├── capability_protocol.go  # Capability / CapabilityManifest
│   ├── context.go              # UnifiedContext（turn-runtime change 扩展字段，此处定骨架）
│   ├── stream.go / stream_bus.go  # 来自 impl-foundation-stream（本 change 只消费）
│   └── events.go               # EventBus 发布接口（EventPublisher）+ CAPABILITY_COMPLETE 常量
└── runtime/
    ├── registry/
    │   ├── tool_registry.go    # ToolRegistry + TOOL_ALIASES + schema 生成
    │   ├── prompt_hints.go     # 5 种 build_prompt_text 格式 + en/zh 文案
    │   └── capability_registry.go
    ├── bootstrap/
    │   └── builtin.go          # BuiltinToolFactories / BuiltinCapabilityFactories 注册点
    └── orchestrator.go         # ChatOrchestrator
```

理由：与 `docs/golang-req/openspec/project.md` 的目录约定一致；`internal/core` 只放纯类型与接口，避免 registry↔tools 的循环依赖（内置工具经 `bootstrap` 的 factory 注册表反向注入，对应基线 `runtime/bootstrap/builtin_capabilities.py` 的 class-path 延迟加载）。

### D2. 关键接口签名（冻结契约）

```go
// internal/core/tool_protocol.go
type ToolResult struct {
    Content       string           `json:"content"`
    Sources       []map[string]any `json:"sources,omitempty"`
    Metadata      map[string]any   `json:"metadata,omitempty"`
    Success       bool             `json:"success"`
    TerminateTurn bool             `json:"terminate_turn,omitempty"`
    PauseForUser  map[string]any   `json:"pause_for_user,omitempty"` // AskUserPayload 形状
}

type ToolDefinition struct {
    Name          string
    Description   string
    Parameters    []ToolParameter
    RawParameters map[string]any // 非 nil 时优先，MCP 工具用
    Deferred      bool
}

type Tool interface {
    Definition() ToolDefinition
    // 工具名与参数命名空间隔离：参数经 map 传递，天然无 positional 冲突（对应基线 positional-only 技巧）
    Execute(ctx context.Context, args map[string]any) (ToolResult, error)
}

// internal/core/capability_protocol.go
type Capability interface {
    Manifest() CapabilityManifest
    Run(ctx context.Context, uc *UnifiedContext, bus *StreamBus) error
}

// internal/runtime/registry/tool_registry.go
func Global() *ToolRegistry // 进程级单例，sync.Once 内做内置加载（单个 factory panic/error 仅告警跳过）
func (r *ToolRegistry) Register(t core.Tool)            // 同名跳过（先注册者保留）
func (r *ToolRegistry) Unregister(name string)          // 不存在为 no-op
func (r *ToolRegistry) Get(name string) (core.Tool, map[string]any, bool) // 返回别名默认参数
func (r *ToolRegistry) GetEnabled(names []string) []core.Tool             // 有序去重、未知跳过
func (r *ToolRegistry) ListTools() []string
func (r *ToolRegistry) DeferredTools() []core.Tool
func (r *ToolRegistry) BuildOpenAISchemas(names []string) []map[string]any // names==nil → 全部
func (r *ToolRegistry) BuildPromptText(names []string, format PromptFormat, language string) (string, error)
func (r *ToolRegistry) Execute(ctx context.Context, name string, args map[string]any) (core.ToolResult, error)

// internal/runtime/orchestrator.go
type ChatOrchestrator struct{ /* capRegistry, toolRegistry, eventPub */ }
// 返回只读事件通道；内部 goroutine 保证通道最终关闭（对应基线 AsyncIterator[StreamEvent]）
func (o *ChatOrchestrator) Handle(ctx context.Context, uc *core.UnifiedContext) <-chan core.StreamEvent
```

`Handle` 用 `<-chan core.StreamEvent` 表达基线的 async generator：orchestrator 内部起一个 goroutine 跑 `capability.Run(uc, bus)`，另一个循环把 bus 订阅通道的事件转发到返回通道，`defer` 链保证「error 事件（如有）→ DONE → close(ch) → bus 注销」的收尾序在任何 panic/cancel 路径下成立（`recover` 把 capability panic 转 `error` 事件）。

### D3. eino 落位

registry 保持框架无关：`Tool.Execute(ctx, map[string]any)` 是唯一执行入口。与 eino 的桥接放在 agent-loop change 的 `einoToolAdapter`（把 `core.Tool` 包成 `tool.InvokableTool`，`InvokableRun` 内做 JSON arguments 解码 → `registry.Execute` → `ToolResult.Content` 序列化），本模块只保证 `BuildOpenAISchemas` 的输出与 eino `schema.ToolInfo` 可无损互转（`raw_parameters` 场景用 eino 的 `openapi3.Schema` 原样承载）。这样 tool 契约不被 eino 的接口形状绑架，CLI/SDK 直接调用工具时也不经过 eino。

### D4. 全局 bus 注册表

`internal/core/stream_bus.go` 提供 `RegisterTurnBus(turnID string, bus *StreamBus)` / `UnregisterTurnBus(turnID)` / `LookupTurnBus(turnID) (*StreamBus, bool)`（`sync.Map`），orchestrator 在 turn_id 非空时注册、收尾时 `defer` 注销——为 api-unified-ws 的 `user_input` 直通输入服务。

### D5. 与 Python 基线映射

| Go | Python 基线 |
| --- | --- |
| `internal/core/tool_protocol.go` | `deeptutor/core/tool_protocol.py`（`BaseTool`/`ToolDefinition`/`ToolResult`） |
| `internal/core/capability_protocol.go` | `deeptutor/core/capability_protocol.py` |
| `internal/runtime/registry/tool_registry.go` | `deeptutor/runtime/registry/tool_registry.py` + `deeptutor/tools/builtin/__init__.py` 的 `TOOL_ALIASES`/`BUILTIN_TOOL_TYPES` |
| `internal/runtime/registry/prompt_hints.go` | tool_registry 的 `build_prompt_text` 五格式 + hints i18n |
| `internal/runtime/registry/capability_registry.go` | `deeptutor/runtime/registry/capability_registry.py` |
| `internal/runtime/bootstrap/builtin.go` | `deeptutor/runtime/bootstrap/builtin_capabilities.py`（class path 表 → factory 函数表） |
| `internal/runtime/orchestrator.go` | `deeptutor/runtime/orchestrator.py`（`ChatOrchestrator.handle`） |

差异说明：基线经字符串 class path + 动态 import 延迟加载，Go 版改为编译期 factory 函数表（`map[string]func() (core.Capability, error)`），「单个加载失败仅告警跳过」语义保留（factory 返回 error / panic 均捕获）。此为实现手段差异，无行为差异，不需登记差异 ID。

### D6. 配置接入（viper）

`max` 轮次等 loop 配置不在本模块；registry 仅经 `internal/config`（impl-foundation-config 的 `runtime_settings`）读取 hint 语言默认值与工具开关类元数据，不直接持有 viper 实例（构造时注入 `config.Provider` 接口），保证单测无文件系统依赖。

## Risks / Trade-offs

- [schema 生成与基线漂移] OpenAI schema 的字段序/缺省值差异会导致严格 provider（Gemini/Anthropic）行为不一致 → 用基线导出的 golden schema JSON 做逐字节对比测试（`BUILTIN_TOOL_TYPES` 全集 + `raw_parameters`/array 边界样例）。
- [channel 版 Handle 的泄漏] 调用方提前放弃消费会卡死转发 goroutine → `Handle` 尊重 `ctx.Done()`，转发 select 双分支；单测用 goroutine 泄漏检测（`goleak`）。
- [单例 + 测试隔离] 进程级 `Global()` 单例污染并行测试 → 全部逻辑挂在可实例化的 `*ToolRegistry` 上，`Global()` 只是薄壳；测试一律 `NewToolRegistry()`。
- [EventBus 尚不存在] cron-events 在 Wave 2 → `EventPublisher` 定义为接口 + no-op 默认实现，orchestrator 发布失败静默容忍（与 spec 一致）。
