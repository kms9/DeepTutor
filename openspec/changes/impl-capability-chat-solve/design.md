# impl-capability-chat-solve — Design

## Context

`chat` 是 deeptutor-go 的默认 capability：一次用户 turn 运行单一 agentic loop（每轮一次 LLM 流式调用 + 并行工具分发），文本实时流向用户；`deep_solve` 不再有独立 pipeline，而是同一 loop 打 `solve_mode` 标记 + 注入 solve system prompt + 追加三件套工具，由内存态 `SolveSession` 提供确定性脊柱。行为事实源为 `docs/golang-req/openspec/specs/capability-chat-solve/spec.md`；loop 原语的细粒度行为（轮结构、mounting、事件时机）在 agent-loop spec 中定义，本 change 负责 capability 级装配与 solve 专属逻辑。约束：前端零改动（stage 事件、result envelope 逐字段对等 M1/M3 fixtures）；技术栈 cloudwego/eino + gin/gorilla（本模块不直接触 HTTP 面）。

## Goals / Non-Goals

**Goals:**
- Go 实现 `chat` 与 `deep_solve` 两个 capability，spec 全部 Requirement/Scenario 对等基线 v1.5.2。
- 提供全项目唯一的 result 收敛点 `EmitCapabilityResult()`，供 mastery/question/research/visualize 等后续 change 复用。
- prompts 资源从基线原样拷贝并实现等价 `{var}` 注入。

**Non-Goals:**
- 不实现 loop 原语本身（轮结构/分发/回退属 impl-agent-loop）；不实现工具（属 impl-tools-builtin）；不实现 turn 生命周期与 waiter（属 impl-turn-runtime）；不实现统一 WS 面（属 impl-api-unified-ws）。
- 不迁移基线中已废弃的独立 solve pipeline（planning/reasoning/writing 三阶段旧实现）——spec 已确认 solve 无独立 stage 事件。

## Decisions

### D1. Go 包结构

```
internal/capabilities/
├── shared/            # 收敛点与跨 capability 公用件
│   ├── result.go      # EmitCapabilityResult()
│   └── prompts.go     # PromptStore：YAML 键寻址 + {var} 注入 + 语言归一
├── chat/
│   ├── capability.go  # ChatCapability：manifest + Run()
│   ├── pipeline.go    # 消息装配、KB seed、工具组合接线、loop 参数（对应 agentic_pipeline.py）
│   ├── loopcap.go     # LoopCapability 接口（solve/mastery 以此挂钩 chat loop）
│   └── prompts/       # {en,zh}/agentic_chat.yaml（基线原样拷贝）
└── solve/
    ├── capability.go  # DeepSolveCapability：manifest + Run()（构造 chat pipeline 复用）
    ├── loopcap.go     # SolveLoopCapability：is_active 判 solve_mode、注入 prompt block、追加三件套
    ├── session.go     # SolveSession + 有界 LRU(256) store
    ├── tools.go       # solve_plan / solve_finish_step / solve_replan
    └── prompts/       # {en,zh}/system.md（基线原样拷贝）
```

### D2. Capability interface 实现签名

沿用 runtime-registry spec 的 `Capability` 协议（Go 侧定义于 `internal/core`）：

```go
type Capability interface {
    Manifest() core.CapabilityManifest
    Run(ctx context.Context, uc *core.UnifiedContext, bus *core.StreamBus) error
}

// chat/capability.go
type ChatCapability struct { /* deps: llm binding factory, tool registry, settings */ }
func (c *ChatCapability) Manifest() core.CapabilityManifest // stages=["exploring","responding"], cli_aliases=["chat"], tools_used=CHAT_OPTIONAL_TOOLS
func (c *ChatCapability) Run(ctx context.Context, uc *core.UnifiedContext, bus *core.StreamBus) error

// solve/capability.go
type DeepSolveCapability struct { chat *chat.ChatCapability }
func (c *DeepSolveCapability) Run(...) error // 置 solve_mode/solve_session_id/solve_max_replans 后委托 chat pipeline

// chat/loopcap.go — solve/mastery 挂钩点
type LoopCapability interface {
    IsActive(uc *core.UnifiedContext) bool
    SystemPromptBlock(uc *core.UnifiedContext, lang string) string
    OwnedTools(uc *core.UnifiedContext) []string
    InjectKwargs(toolName string, uc *core.UnifiedContext) map[string]any // 服务端私有参数（_solve_session_id 等）
    Exclusive() bool
}
```

### D3. 阶段编排落位：自定义 loop 函数，不用 eino Graph（取舍结论）

- **结论**：chat 的合并 loop 用**自定义阶段函数**（`runAgenticLoop`，即 impl-agent-loop 交付的原语）实现，**不用 eino Graph/Chain 编排**，也不用 eino 预置 ReAct agent。
- **理由**：(1) spec 要求的行为高度定制——narration/finish 的 `call_role` 标记、空 finish nudge、`_context_checkpoint` 折叠、ask_user 暂停后**原地替换** `role=tool` 消息、预算耗尽的禁工具收尾调用——这些都要求对消息序列做过程内可变操作，eino Graph 的不可变 state 传递模型表达不经济；(2) chat 本来只有一个 stage（`responding`），没有可静态编排的多阶段拓扑，Graph 带来的可视化/重试收益为零。
- **eino 的角色**：LLM 调用统一走 llm-provider 提供的 eino `model.ToolCallingChatModel`（`Generate`/`Stream` + `WithTools`）；流式经 `schema.StreamReader[*schema.Message]` 由 loop 侧映射到 StreamEvent。即「eino 管调用，Go 手写代码管编排」。多阶段 pipeline 的 capability（research/question）同样采纳该结论，各自 design 单独论证。

### D4. stage 事件发射点

- `bus.Stage(ctx, "responding", source="chat", fn)` 包裹整个 loop：进入发 `stage_start(stage="responding", source="chat")`，退出（含 panic/err path）发 `stage_end`。`exploring` 仅存在于 manifest 声明，**永不发事件**（名实分离，前端兼容）。
- solve turn 完全相同——manifest `stages=["responding"]`，不发 planning/reasoning/writing 任何 stage 事件（spec 有意差异说明）。
- loop 内事件（content/thinking/tool_call/tool_result/progress/sources）发射时机由 agent-loop spec 定义，本 change 只保证 stage 包裹与最终 result 次序（聚合 sources → result → 由 orchestrator 补 DONE）。

### D5. 结果收敛点

`shared.EmitCapabilityResult(bus, source, payload, usage *UsageTracker)`：payload 为 `map[string]any`（chat：`{response, completed, engine:"agent_loop", rounds, tool_steps}`）；usage 有记录时把 `usage.Summary()` 合入 `payload.metadata.cost_summary`（保留已有 metadata 键）。所有 capability 一律经此函数发 result（deep_research 的 outline_preview 是登记在其 spec 的唯一例外，见该 change）。

### D6. prompts 资源加载

- 采用 **`go:embed`** 把 `internal/capabilities/{chat,solve}/prompts/{en,zh}/*` 编入二进制（单二进制分发是 cli-launcher 的目标形态），目录组织与基线 `{en,zh}` 完全一致、文件内容逐字节拷贝自基线。
- `shared.PromptStore`：`Get(lang, keyPath string, vars map[string]any, fallback string) string`——YAML 键路径寻址（`notices.context_window_guard`）；`{var}` 注入实现 Python `str.format(**kwargs)` 的命名占位符子集（不支持位置参数与格式说明符，基线未使用）；键缺失/注入失败回退 `fallback` 不报错；语言归一 `zh*` → `zh`，其余 `en`。solve system.md 整文件读取，允许被 chat prompts 的 `solve.system` 键覆盖。

### D7. SolveSession 状态机

进程内 `map[string]*SolveSession` + 容量 256 的 LRU（`container/list` 实现，写锁保护）；字段 `Analysis`、`Steps []SolveStep{ID, Goal, Done, Summary}`、`Replans`、`MaxReplans`。三件套为普通 `BaseTool` 实现但**不进全局注册表的用户可见面**，仅由 SolveLoopCapability 追加；服务端经 `InjectKwargs` 注入 `_solve_session_id` 与 `max_replans`。`solve_finish_step` 成功时在 ToolResult metadata 附 `_context_checkpoint.summary`，触发 loop 折叠（折叠机制在 agent-loop）。不持久化：计划以工具结果 JSON 留存在会话消息里。

### D8. 与 Python 基线文件映射表

| Python 基线 | Go 落位 |
| --- | --- |
| `deeptutor/agents/chat/capability.py` | `internal/capabilities/chat/capability.go` |
| `deeptutor/agents/chat/agentic_pipeline.py` | `internal/capabilities/chat/pipeline.go` |
| `deeptutor/agents/chat/agent_loop.py` | `internal/agentloop/`（impl-agent-loop 交付，本 change 消费） |
| `deeptutor/agents/chat/prompt_blocks.py` | `internal/capabilities/chat/pipeline.go`（prompt block 组装函数） |
| `deeptutor/agents/chat/prompts/{en,zh}/agentic_chat.yaml` | `internal/capabilities/chat/prompts/{en,zh}/agentic_chat.yaml`（原样拷贝） |
| `deeptutor/agents/_shared/capability_result.py` | `internal/capabilities/shared/result.go` |
| `deeptutor/capabilities/solve/capability.py` | `internal/capabilities/solve/capability.go` |
| `deeptutor/capabilities/solve/loop.py` | `internal/capabilities/solve/loopcap.go` |
| `deeptutor/capabilities/solve/tools.py` | `internal/capabilities/solve/tools.go` |
| `deeptutor/capabilities/solve/session.py` | `internal/capabilities/solve/session.go` |
| `deeptutor/capabilities/solve/prompts/{en,zh}/system.md` | `internal/capabilities/solve/prompts/{en,zh}/system.md`（原样拷贝） |

## Risks / Trade-offs

- [Python `str.format` 语义差异（嵌套花括号 `{{}}` 转义、缺键 KeyError）] → PromptStore 只实现基线 prompts 实际用到的子集；对全部基线 prompts 文件跑一遍「渲染快照测试」与 Python 输出 diff。
- [eino StreamReader 到 StreamEvent 的 chunk 边界与基线不一致导致 fixtures diff] → fixtures 对比按「拼接后文本 + 事件类型序列」对齐而非逐 chunk 字节对齐（acceptance.md 3.1 白名单机制）；`call_status`/`sources`/`result` 等离散事件必须逐字段一致。
- [SolveSession 为进程内状态，多副本/重启即失] → 与基线一致（有意同构，spec 已声明不持久化），无需缓解；文档中明示。
- [chat 是最大扇出依赖，接口一旦变动波及 4+ 个 capability change] → `LoopCapability` 接口在本 change 内冻结并在 tasks 中先行评审；后续 change 只增不改。

## Migration Plan

无线上迁移；按 tasks 顺序实现，M1 交付 chat（fixtures：普通 chat turn / ask_user / cancel / regenerate），M3 补 solve。回滚 = 不注册 capability。

## Open Questions

- `UsageTracker` 的成本汇总字段精度（浮点格式化）是否需要与 Python 输出逐字符一致——待 M1 fixtures 录制后确认，必要时登记白名单。
