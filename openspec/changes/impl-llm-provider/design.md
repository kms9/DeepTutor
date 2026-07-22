# Design: impl-llm-provider

## Context

基线 LLM 层由三块组成：`deeptutor/services/provider_registry.py`（30+ provider 元数据、别名、匹配与探测）、`deeptutor/services/llm/provider_core/`（`openai_compat_provider.py`、`anthropic_provider.py` 两类协议内核 + `base.py` 的 `LLMResponse` 归一）、`deeptutor/core/agentic/client.py` + `usage.py`（agentic 侧消费与 UsageTracker）。Go 版技术选型固定为 `cloudwego/eino` + `eino-ext`：provider 适配实现为 eino `ChatModel`，流式经 `StreamReader[*schema.Message]` 输出，由 agent-loop 映射到 `StreamEvent`。契约测试要求离线确定性（acceptance.md §2），因此需要 replay provider（Go 版新增测试设施，spec 已注明基线无此物）。

## Goals / Non-Goals

**Goals:**

- 注册表/别名/匹配探测与基线一一对应；两类内核归一到同一响应与 chunk 形状。
- 请求侧兼容全集：消息清洗、`max_tokens`/`max_completion_tokens` 切换、thinking 注入、`response_format` 降级记忆、model_overrides。
- 重试/图片降级、UsageTracker、mock/replay provider。
- OAuth 两家（`openai_codex`/`github_copilot`）注册表可见但调用报「后置」错误。

**Non-Goals:**

- agentic loop 本体与 tool 执行（归 agent-loop）；`cost_summary` 注入结果事件的动作由 capability 收尾逻辑执行（本模块只提供 `Summary()`）。
- `llm_configs.json`/`model_catalog.json` 的 REST 管理面（归 settings-api）；本模块只读取绑定配置。
- OAuth 协议实现（M2 后单独排期）。

## Decisions

### D1. Go 包结构

```
deeptutor-go/internal/llm/
├── registry/          # provider 注册表、别名、匹配与探测
│   ├── spec.go        # ProviderSpec 元数据结构 + 内置表（注册顺序即优先级）
│   ├── canonical.go   # CanonicalProviderName + 别名表
│   └── match.go       # FindByName / FindByModel / FindGateway / StripProviderPrefix
├── core/              # 协议无关的归一层
│   ├── response.go    # LLMResponse / ToolCall / Usage
│   ├── jsonrepair.go  # malformed arguments 容错修复
│   ├── sanitize.go    # 消息清洗（键白名单、空 content、tool_call id 归一）
│   └── retry.go       # 瞬态重试 + 图片降级
├── openaicompat/      # OpenAI chat.completions SSE 内核（eino ChatModel）
├── anthropic/         # Anthropic Messages 内核 + OpenAI 形状适配
├── replay/            # mock/replay ChatModel（仅测试）
├── usage/             # UsageTracker
└── factory.go         # NewChatModel(binding) 按 backend 分派
```

### D2. eino 落位：内核实现为 `model.ToolCallingChatModel`

```go
package llm

import (
    "github.com/cloudwego/eino/components/model"
    "github.com/cloudwego/eino/schema"
)

type Binding struct { // 来自 llm_configs.json（经 impl-foundation-config 读取）
    Provider string // canonical 名
    Model    string
    APIKey   string
    APIBase  string
    Extra    map[string]any // reasoning_effort、thinking 开关等
}

// factory：按注册表 backend 分派内核；OAuth backend 返回 ErrProviderDeferred
func NewChatModel(ctx context.Context, b Binding) (model.ToolCallingChatModel, error)
```

- **openai_compat / azure_openai** backend：以 `eino-ext` 的 `components/model/openai` 组件承载 HTTP/SSE 传输，外层包 `normalizingChatModel` 装饰器（同样实现 `model.ToolCallingChatModel`）完成基线特有语义：请求前清洗与参数注入（D4）、响应/chunk 归一（D3）、`response_format` 降级重试与 (binding, model) 记忆（进程内 `sync.Map`）。
- **anthropic** backend：以 `eino-ext` 的 claude 组件承载传输，适配层复刻基线 `_ProviderOpenAIAdapter`/`_ProviderOpenAIStream`：内容增量实时透出；`input_json_delta` 累积的工具调用在流结束后以带 `Index` 的 tool_call chunk 依次补发；最后补一帧带 `FinishReason` + `Usage` 的收尾 chunk；`supports_prompt_caching` 时按基线策略在 system/tools 放 `cache_control`。消费方拿到的 `StreamReader[*schema.Message]` 序列与 OpenAI-compat 内核结构一致（spec「Anthropic 流对消费方透明」）。
- chunk 载体统一为 `*schema.Message`：内容增量走 `Content`、思考增量走 reasoning 扩展字段、tool_call 分片走 `ToolCalls[].Index`（`*int`）、收尾帧走 `ResponseMeta.FinishReason` + `ResponseMeta.Usage`。跨内核 chunk→`LLMResponse` 的聚合用同一个 `Accumulate(stream *schema.StreamReader[*schema.Message]) (LLMResponse, error)`。

### D3. tool_call 分片归组与 json repair

`Accumulate` 按 `Index` 归组：`ID`/`Name` 取首个非空、`Arguments` 字符串逐段拼接；流结束后 `jsonrepair.Parse(argStr)`——先 `json.Unmarshal`，失败则容错修复（截尾补括号、单引号、尾逗号等基线 json_repair 同级能力，选用 Go 移植库或自研最小实现），仍失败回落 `{}` 不报错。`LLMResponse`：

```go
type LLMResponse struct {
    Content          string
    ToolCalls        []ToolCall // {ID, Name string; Arguments map[string]any; ProviderSpecific map[string]any}
    FinishReason     string     // 默认 "stop"
    Usage            Usage      // {PromptTokens, CompletionTokens, TotalTokens int}
    ReasoningContent string
}
```

### D4. 请求侧兼容（sanitize.go + 各内核请求构造）

- 消息键白名单（`role`/`content`/`tool_calls`/`tool_call_id`/`name`/`reasoning_content` 等）；空 content：assistant 且带 tool_calls → null，否则 `"(empty)"`；多模态数组剔除空 text 分片；tool_call id 按 provider 约束归一且映射跨消息一致（进程内每请求映射表）。
- `supports_max_completion_tokens` 决定 `max_tokens` vs `max_completion_tokens`；`thinking_style` 决定 `thinking_type`/`enable_thinking`/`reasoning_split` 注入方式；`model_overrides`（如 Moonshot `kimi-k2.5`/`kimi-k2.6` 强制 `temperature=1.0`）在请求构造最后一步套用；`strip_model_prefix` 时剥 `provider/` 前缀。
- 重试包装器 `retry.go`：错误文案命中瞬态特征集 → 1s/2s/4s 退避；超次返回 `LLMResponse{FinishReason:"error"}` 不抛异常；非瞬态 + 含图片 + 允许降级 → 剥图重试一次。

### D5. UsageTracker（internal/llm/usage）

```go
type UsageTracker struct{ mu sync.Mutex; calls int; prompt, completion, total int64; costUSD float64 }
func (t *UsageTracker) AddFromResponse(model string, u Usage) // 精确路径，计 1 次调用
func (t *UsageTracker) AddEstimated(model string, chars int)  // chars/3.5 折算
func (t *UsageTracker) Summary() map[string]any               // 零调用返回 nil；否则五字段
```

流式记账约定由 `Accumulate` 保证：只保留最后一帧 usage、流结束提交一次；无 usage 帧回落字符估算；记账 panic/错误吞掉不影响已完成的流。定价表以每千 token 计价（随 model_catalog 数据）。`metadata.cost_summary` 注入由 capability 收尾（`emit_capability_result` 对等物）调用 `Summary()` 完成。

### D6. mock/replay provider（internal/llm/replay）

自定义 `model.ToolCallingChatModel`：从 `contracts/fixtures/llm/*.jsonl` 加载录制 chunk 序列（内容增量、reasoning 增量、分片 tool_call、usage 帧、finish 帧），`Stream` 用 `schema.Pipe` 逐 chunk 保序写入 `StreamReader`，经与真实内核**相同**的 `Accumulate` 归一路径输出。fixture 消费不完整或多余 chunk 时由 `Fixture.AssertExhausted(t)` 使断言可见。注册方式：仅测试构造器 `replay.New(fixturePath)` 直接注入，`factory.go` 的生产分派表不含该 backend（SHALL NOT 进生产选择路径）。

### D7. 与 Python 基线的映射表

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/services/provider_registry.py` | `internal/llm/registry/` |
| `services/llm/provider_core/base.py` `LLMResponse` | `internal/llm/core/response.go` |
| `provider_core/openai_compat_provider.py` | `internal/llm/openaicompat/`（eino-ext openai + 归一装饰器） |
| `provider_core/anthropic_provider.py` + `_ProviderOpenAIAdapter` | `internal/llm/anthropic/` |
| `services/llm/executors.py`/`client.py` 清洗与重试 | `internal/llm/core/sanitize.go` + `retry.go` |
| `deeptutor/core/agentic/usage.py` `UsageTracker` | `internal/llm/usage/` |
| 基线测试的脚本化注入客户端（无独立 replay provider） | `internal/llm/replay/`（Go 新增，spec 已登记为测试设施） |

## Risks / Trade-offs

- [eino-ext 组件不透出个别底层字段（如 `delta.reasoning` 变体、非标 usage 帧）] → 归一装饰器持有原始 SSE 帧的扩展字段入口；若某 gateway 变体确实穿不透，openaicompat 内核保留「自持 HTTP+SSE 解析」的降级实现路径，对外 ChatModel 形状不变。
- [json repair 的 Go 实现与 Python json_repair 修复结果不完全一致] → 契约以「修复成功为 object、失败回 `{}` 不报错」为准（spec 语义），并用 fixtures 固定高频 malformed 样例回归。
- [(binding, model) 的 response_format 记忆是进程内状态，重启后丢失] → 与基线同为进程内记忆，行为对等；首次失败重试一次的成本可接受。
- [定价表漂移导致 `total_cost_usd` 与基线不同] → 定价数据与 `model_catalog.json` 同源（foundation-config 读取），差异属数据而非行为。
- [OAuth 后置可能被误接生产路径] → factory 对 `openai_codex`/`github_copilot` 返回哨兵错误 `ErrProviderDeferred`，错误文案指明后置项（spec「选择 Codex 得到明确错误」），并有测试锁定。

本模块无 acceptance.md §6 登记编号的有意差异（D-001~D-008 不涉及 llm-provider）；mock/replay provider 为 Go 版新增测试设施，其「基线无此物」的说明已固化在模块 spec 正文中。
