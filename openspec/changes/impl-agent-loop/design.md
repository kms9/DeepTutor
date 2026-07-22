## Context

agent-loop 是 chat capability 的执行引擎（Wave 1 / M1），基线实现为 `deeptutor/agents/chat/agentic_pipeline.py`（约 1.2k 行）+ `agent_loop.py` + `core/agentic/tool_dispatch.py`。它不是普通的 ReAct 循环：带有轮次预算强制收敛、narration/finish 双角色文本、同批 tool_call 去重 stub、`ask_user` 轮中暂停/恢复（替换 `role=tool` 消息内容后续跑）、上下文窗口守护、provider 兼容性回退等大量与 StreamEvent 协议耦合的语义。技术约束：必须落在 cloudwego/eino + eino-ext 上（ChatModel / StreamReader / tool 抽象），事件面必须与 WS golden fixtures 逐事件对等。上游：`impl-llm-provider`（eino ChatModel + mock/replay）、`impl-runtime-registry`（ToolRegistry）、`impl-foundation-stream`（StreamEvent/StreamBus）。

## Goals / Non-Goals

**Goals:**

- 与基线逐事件对等的单 Agent Loop（mock provider 回放下 fixtures 全过）。
- 用 eino 的 ChatModel/StreamReader/ToolInfo 抽象承载 LLM 交互，保持 provider 无关。
- 工具挂载组装（context-gated mounting）与分发（并行、去重、kwargs 注入）行为对等。
- `ask_user` 暂停/恢复的 loop 侧完整语义（runtime 侧 waiter 由 turn-runtime 提供）。

**Non-Goals:**

- 不实现具体工具（tools-builtin）与 chat capability 的 explore 前置 pass / prompt 资产（capability-chat-solve）。
- 不实现 runtime 侧回复队列、turn 状态机（turn-runtime）。
- 不实现 MCP 客户端本体（deferred 池对接 registry 的 `DeferredTools()`，MCP 接入在后续 change）。

## Decisions

### D1. eino ReAct agent vs 自定义 loop —— 结论：自定义 loop，eino 只用组件层

**评估**：eino 的预制 `react.Agent`（`flow/agent/react`）提供 ChatModel→ToolsNode 的 graph 循环，但其循环控制点与基线语义冲突：

| 基线语义 | eino ReAct agent 支持度 |
| --- | --- |
| 轮次预算耗尽后**关闭工具**再调一次收敛（消息还要注入收敛指令） | ✗ MaxStep 到达即报错终止，无「降级收尾调用」钩子 |
| 每轮文本实时流出并按 `call_role: narration/finish` 打标 | ✗ 中间轮消息在 graph 内部流转，无 per-round 事件回调面 |
| 同批 tool_call 去重：重复项不执行、以 stub 文本回写、`ask_user` 重复项还要抑制事件 | ✗ ToolsNode 全量执行，无分发前拦截点 |
| `ask_user` 暂停：await 外部 waiter、把回复**替换**进对应 `role=tool` 消息再续跑 | ✗ graph 单次运行不可中途挂起等外部输入再改写历史消息 |
| 每轮调用前窗口守护（snip 最早 tool 消息）与 `_context_checkpoint` 折叠 | ✗ 无 per-round 消息重写钩子 |
| provider 回退（去 stream_options / 去 tools / 剥图片重试） | ✗ 需要包裹在模型调用点外层 |

以上每条都是 spec 的硬需求，逐条 hack eino graph 的成本高于收益。**结论**：自定义 `for` 循环（对应基线 `agent_loop.py` 本就是显式循环），复用 eino 的**组件层**——`model.ToolCallingChatModel`（`Generate`/`Stream`、`WithTools` 绑定 `*schema.ToolInfo`）、`schema.StreamReader[*schema.Message]`（流式 delta 消费）、`schema.Message`/`ToolCall`（消息与 tool_call 归一形状）。工具经 `einoToolAdapter` 实现 `tool.InvokableTool`（Info 来自 registry schema、InvokableRun 转发 `registry.Execute`），保证未来接 eino 生态（callbacks、Langfuse tracing）零改造；但**分发不经 ToolsNode**，由自研 dispatcher 控制并行/去重/stub/pause。

### D2. Go 包结构

```
internal/
├── agents/chat/
│   ├── pipeline.go        # AgenticChatPipeline：Run(ctx, uc, bus)（实现 core.Capability 的执行部分）
│   ├── loop.go            # runLoop：轮循环、预算、收敛、nudge、ask_user 暂停恢复
│   ├── messages.go        # 会话组装（system prompt / 摘要 header / 历史 / user 消息 + KB seed 拼接）
│   ├── mounting.go        # composeEnabledTools 四步组装 + allowed/forced/suppressed/exclusive 裁剪
│   ├── deferred.go        # deferred 池准备、load_tools 会话级已加载集、PageIndex 预加载
│   ├── seed.go            # KB seed 并行预检索
│   └── events.go          # call_status/tool_call/tool_result/sources/result 事件产生
└── llm/agentic/
    ├── dispatch.go        # DispatchToolCalls：并行（≤8）、判重、stub、kwargs 注入、子 trace
    ├── stream_relay.go    # StreamReader → StreamEvent（content/thinking 增量、think 标签分流状态机）
    ├── window_guard.go    # token 估算、snip、_context_checkpoint 折叠
    ├── fallback.go        # provider 兼容性回退（stream_options / tools / 图片）
    └── eino_tool.go       # einoToolAdapter（core.Tool → tool.InvokableTool）
```

### D3. 关键接口签名

```go
// internal/agents/chat/pipeline.go
type AgenticChatPipeline struct { /* model llm.ChatModelProvider; registry *registry.ToolRegistry; cfg config.Provider; usage *llm.UsageTracker */ }
func (p *AgenticChatPipeline) Run(ctx context.Context, uc *core.UnifiedContext, bus *core.StreamBus) error
func (p *AgenticChatPipeline) EffectiveMaxRounds(uc *core.UnifiedContext) int // max(cfg max_rounds≥1, metadata["_min_loop_rounds"])

// internal/agents/chat/mounting.go
func ComposeEnabledTools(uc *core.UnifiedContext, reg *registry.ToolRegistry, gates MountGates) []string
type MountGates struct { HasRagKB, HasL3Memory, HasNotebook, HasSkills, HasDeferred, SandboxAllowed bool }

// internal/llm/agentic/dispatch.go
type DispatchResult struct {
    ToolMessages []*schema.Message      // 与 tool_calls 一一配对的 role=tool 消息（含 stub）
    Pause        *PausePayload          // 首个 pause_for_user 胜出
    Terminate    bool                   // terminate_turn
    Sources      []map[string]any
}
func DispatchToolCalls(ctx context.Context, calls []schema.ToolCall, reg *registry.ToolRegistry,
    inject KwargsInjector, sink EventSink) (DispatchResult, error)

// internal/llm/agentic/stream_relay.go
// 消费 eino StreamReader，增量拆分 think 标签，经 sink 发 content/thinking 事件，返回完整 assistant 消息
func RelayModelStream(ctx context.Context, sr *schema.StreamReader[*schema.Message],
    sink EventSink, callID string) (*schema.Message, error)

// ask_user waiter（turn-runtime 经 UnifiedContext.Metadata["wait_for_user_reply"] 注入）
type UserReplyWaiter func(ctx context.Context) (*UserReply, bool) // false = 无回复（取消/放弃）
type UserReply struct { Text string; Answers []QuestionAnswer }   // v2 [{questionId, text}]
```

轮循环骨架（`loop.go`）：组装消息 → 窗口守护 → `model.WithTools(schemas)` + `Stream` → `RelayModelStream`（call_status running/complete 包裹）→ 无 tool_call 且文本非空 → finish；有 tool_call → `DispatchToolCalls` → pause 则 await waiter（回复替换匹配 `pause_tool_call_id` 的 tool 消息 + `ask_user_resolved` progress）→ 追加消息继续；轮次耗尽/中途失败 → 收敛调用（不 `WithTools`）。

### D4. 与 Python 基线映射

| Go | Python 基线 |
| --- | --- |
| `internal/agents/chat/pipeline.go` / `loop.go` | `deeptutor/agents/chat/agentic_pipeline.py`（`AgenticChatPipeline.run`）、`agent_loop.py` |
| `internal/agents/chat/mounting.go` | `agentic_pipeline._compose_enabled_tools` + `agents/_shared/tool_composition.py` |
| `internal/agents/chat/messages.go` | `_build_loop_messages` / `_build_system_prompt` / `_prepare_messages_with_attachments` |
| `internal/agents/chat/seed.go` | `_retrieve_kb_seed_block` / `_seed_search_one_kb` |
| `internal/agents/chat/deferred.go` | `_prepare_deferred_tools` / `_deferred_tools_manifest` / `_pageindex_doc_maps` |
| `internal/llm/agentic/dispatch.go` | `core/agentic/tool_dispatch.py` + `_dispatch_tool_calls` / `_augment_tool_kwargs` |
| `internal/llm/agentic/stream_relay.go` | agentic client 的流式 delta 处理 + think 标签分流 |
| `internal/llm/agentic/window_guard.go` | `_guard_context_window` / `_estimate_messages_tokens` |
| `internal/llm/agentic/fallback.go` | client 的 stream_options / tool schema / 图片降级重试 |

### D5. 并发模型

工具并行用 `errgroup.WithContext` + 信号量（上限 8），每个工具 goroutine 只写自己的结果槽（按 index 预分配，保证 tool 消息与 tool_calls 顺序配对），事件经 `EventSink`（薄封装 `bus.Emit`，StreamBus 内部串行化 seq）发出。`ask_user` waiter 是阻塞点：`waiter(ctx)` 同时 select turn ctx 取消，保证 cancel_turn 能解除等待（语义与 turn-runtime spec 的「取消解除 ask_user 等待」衔接）。

### D6. 差异说明

本模块无新增有意差异；`ask_user` 暂停期间的 turn 状态展示涉及 turn-runtime 的 **D-002**（`waiting_input`），loop 侧仅经 waiter 接口感知，不落新差异。事件时序、stub 文案等以 fixtures 对等为准。

## Risks / Trade-offs

- [eino 版本演进] eino 的 `ToolCallingChatModel`/`StreamReader` API 仍在迭代 → go.mod 锁定版本；适配面收敛在 `llm-provider` change 的 `ChatModelProvider` 接口后，loop 不直接 import eino-ext。
- [think 标签状态机边界] 跨 chunk 标签、嵌套/未闭合标签易与基线漂移 → 从基线单测样例移植 golden 用例（含 `</think` 断切、无闭合冲刷）。
- [事件顺序竞态] 工具并行执行时 tool_result 事件次序可能与基线不同 → 按基线语义：tool_call 事件按 tool_calls 原序在分发前逐个发出，tool_result 按完成序发出（fixtures 录制即如此），测试用确定性 mock 工具固定完成序。
- [窗口估算差异] Go 侧 token 估算器与基线 tiktoken 计数有偏差，snip 触发点可能不同 → 估算器用同一「字符/4 + 消息开销」近似并以基线样例校准；snip 行为本身有 warning progress 可观测。
- [pause 挂起泄漏] waiter 永不返回会挂死轮 goroutine → waiter 必须绑定 turn ctx；turn-runtime 收尾时关闭队列使 waiter 返回 `false`。
